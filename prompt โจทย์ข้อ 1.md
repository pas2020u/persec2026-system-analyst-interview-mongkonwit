สร้าง PDF ให้ทีตามข้อมูลดังนี้
บริษัทกำลังจะจัดแคมเปญ "Flash Sale" แจกโค้ดส่วนลด 1,000 สิทธิ์ โดยจะเปิดให้กดพร้อมกันเวลา 12:00 น. คาดว่าจะมีผู้ใช้งาน (Traffic) เข้ามาพร้อมกันบนระบบทั้ง Web และ Mobile App ประมาณ 50,000 คนภายใน 5 นาทีแรก
คําถาม:
1. ในฐานะ SA ให้ออกแบบ High-Level System Architecture สำหรับระบบนี้ คุณจะใช้ Tech Stack หรือ Component ใดบ้าง (เช่น Database, Cache, Message Queue ฯลฯ) เพื่อให้ระบบไม่ล่ม?
2. คุณจะออกแบบ Logic อย่างไรเพื่อแก้ปัญหา Concurrency (Race Condition) เพื่อป้องกันไม่ให้คนกดรับโค้ดส่วนลดซ้ำซ้อนหรือแจกโค้ดเกิน 1,000 สิทธิ์?

คำตอบ จะใช้ Tech stack และ Component ดังนี้
1.Component Edge & Traffic Control ใช้ Tech Stack Cloudflare ไว้แคชหน้า Static ไว้ที่ Edge 100% เพื่อไม่ให้เซิร์ฟเวอร์เสีย CPU ส่งหน้าเว็บ และตั้ง Rate Limit กัน Bot
2.Component  API Gateway  ใช้ Tech Stack Nginx Ingress ทำ Per-User Rate Limiting 1 User ID กดได้ไม่เกิน 1 ครั้งต่อวินาทีและตรวจสอบ JWT Signature 
3.Component Application Service ใช้ Tech Stack  Node.js ใช้เนื่องจาก Concurrent Connections สูงและกิน Memory น้อย ตั้งค่า Pre-scale Pods รอไว้ล่วงหน้าได้
4.Component In-Memory  ใช้ Tech Stack Redis Cluster ใช้เช็กและตัดสิทธิ์จบใน RAM รวดเร็ว
5.Component  Message Broker ใช้ Tech Stack RabbitMQ ทำหน้าที่เป็น Shock Absorber รับข้อมูลเฉพาะ 1,000 คนที่ได้สิทธิ์ ส่งต่อไปบันทึกลง DB ตามจังหวะที่ DB รับไหว
6.Component   Database ใช้ Tech Stack MySQL  MySQL InnoDB)เก็บข้อมูล Transaction  โดยรับโหลดจาก Worker

การออกแบบ Logic แก้ปัญหา Concurrency
สำหรับการแก้ปัญหา Race Condition ในฝั่ง Node.js คู่กับ Redis ห้ามเขียนโค้ดแบบสั่ง await redis.get() มาเช็กค่าใน if แล้วค่อยสั่งหักลบแยกบรรทัด โดยจะกำหนดการล็อกไม่ให้คนเดิมกดซ้ำด้วย redis.sadd() แทนที่เราจะไปเช็กว่าเขาเคยรับหรือยังเราจะยิง await redis.sadd() เพื่อเอา user_id ของเขายัดใส่ Redis Set ถ้าคนนี้เพิ่งกดครั้งแรก Redis จะใส่ค่าสำเร็จแล้วตอบกลับมาเป็น 1 เราก็ให้ไปต่อ แต่ถ้าเขาเปิดหลายแท็บหรือเขียนสคริปต์ยิงรัวเข้ามาพร้อมกัน เธรดที่ตามมาจะได้ค่า 0 กลับไปถ้าเขาเคยกดมาแล้วเราก็ตอบกลับได้เลยว่า 'คุณรับสิทธิ์ไปแล้ว'  ต่อด้วยตัดโควตา 1,000 สิทธิ์ ด้วย redis.decr() สั่ง await redis.decr() หักลบเลขโควตาใน Redis ลงไป 1  สิทธิ์คำสั่ง DECR จะเรียงคิวหักลบทีละคน ถ้าหักแล้วได้เลขตั้งแต่ 999 ลงมาจนถึง 0 แปลว่าเขาทัน 1,000 คนแรกก็ส่ง user_id เข้า Queue บันทึกลง Database ถ้าคนที่ 1,001 กดเข้ามา เลขที่ได้กลับมาจะกลายเป็น -1 แปลว่าสิทธิ์เต็มแล้วให้สั่ง Rollback คืนค่า แล้วตั้งตัวแปร Flag ใน Memory ของ Node.js ว่า isSoldOut = true โดยสามารใช้ไลบรารี ioredis ทำขั้นตอนเช็กชื่อและหักโควตาไว้ใน redis.defineCommand() ได้เพื่อเพิ่มประสิทธิภาพของระบบได้