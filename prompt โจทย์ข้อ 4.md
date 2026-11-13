ทำ PDF ไฟล์ตามข้อมูลดังนี้
สถานการณ์ที่ 4
หากคุณพบว่ามีคนฝัง API Key หรือ URL ของ 3rd-party ไว้ใน Source Code ตรงๆ (Hardcode) คุณจะอธิบายผลกระทบที่จะตามมาให้เขาฟังอย่างไร และจะให้เขาแก้วิธีเก็บ Key อย่างไรให้ถูกต้อง?

คำตอบ อธิบายให้เห็นภาพความเสียหายจริงถ้าหลุดขึ้น Git เท่ากับหลุดถาวร เสี่ยงโดนสวมรอยใช้งานจนเกิดค่าใช้จ่ายมหาศาลหรือข้อมูลรั่วไหลฝัง URL ทำให้แยก Environment (Dev/UAT/Prod) ไม่ได้ และถ้า 3rd-party เปลี่ยน URL เราต้องแก้โค้ด Build ใหม่และ Deploy ใหม่ทั้งหมด แก้โดยแยก Configuration ออกจาก Code (ตามหลัก 12-Factor App) โดยใช้ Environment Variables สำหรับ URL และใช้ Secret Manager สำหรับ API Key ประเภทข้อมูล 3rd-Party URL (ไม่เป็นความลับ)เครื่องมือที่ใช้เก็บ ไฟล์ .env หรือไฟล์ Config ประจำ Environment การเรียกใช้ในโค้ด process.env.THIRD_PARTY_API_URL และประเภทข้อมูลAPI Key / Secret (ความลับสูงสุด) เครื่องมือที่ใช้เก็บไฟล์ .env บนเครื่องตัวเอง (ที่อยู่ใน .gitignore เท่านั้น)ดึงผ่าน Secret Manager SDK หรือฉีดเข้า Env ตอนรัน Container
