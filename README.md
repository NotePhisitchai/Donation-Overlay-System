<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OBS Donation Overlay</title>
    <!-- Tailwind CSS (CDN) -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Google Fonts: Kanit -->
    <link href="https://fonts.googleapis.com/css2?family=Kanit:wght@300;400;600;700&display=swap" rel="stylesheet">
    
    <style>
        /* ตั้งค่าพื้นหลังให้โปร่งใสสำหรับ OBS */
        body {
            background-color: transparent !important;
            font-family: 'Kanit', sans-serif;
            margin: 0;
            padding: 0;
            overflow: hidden; /* ป้องกัน Scrollbar ปรากฏใน OBS */
        }

        /* Animation สำหรับ Alert */
        .alert-enter {
            animation: bounceIn 0.8s cubic-bezier(0.25, 1, 0.5, 1) forwards;
        }
        .alert-exit {
            animation: slideUpOut 0.5s ease-in forwards;
        }

        @keyframes bounceIn {
            0% { transform: translateY(-150%) scale(0.8); opacity: 0; }
            60% { transform: translateY(10%) scale(1.05); opacity: 1; }
            100% { transform: translateY(0) scale(1); opacity: 1; }
        }

        @keyframes slideUpOut {
            0% { transform: translateY(0); opacity: 1; }
            100% { transform: translateY(-150%); opacity: 0; }
        }

        /* เอฟเฟกต์แสงวิบวับรอบกล่อง Alert */
        .glow-effect {
            box-shadow: 0 0 20px rgba(var(--theme-color-rgb), 0.6);
        }
    </style>
</head>
<body class="w-screen h-screen relative">

    <!-- ============================== -->
    <!-- 1. ส่วนแสดง QR Code พร้อมเพย์  -->
    <!-- ============================== -->
    <!-- สามารถปรับ position ได้โดยแก้คลาส bottom-8, right-8 -->
    <div id="qr-container" class="absolute bottom-8 right-8 bg-white/95 backdrop-blur-sm p-4 rounded-2xl shadow-xl flex flex-col items-center border-4 border-blue-500 transition-all hover:scale-105">
        <h3 class="text-blue-700 font-bold text-lg mb-2">สนับสนุนสตรีมเมอร์</h3>
        <!-- ใช้ API สร้าง QR Code ของ PromptPay (ใส่เบอร์ใน JS) -->
        <img id="promptpay-qr" src="" alt="PromptPay QR" class="w-32 h-32 object-cover rounded-lg">
        <p class="text-gray-600 text-sm mt-2 font-semibold">สแกนเพื่อโดเนท</p>
    </div>

    <!-- ============================== -->
    <!-- 2. ส่วนแจ้งเตือน (Alert Box)   -->
    <!-- ============================== -->
    <!-- ซ่อนเป็นค่าเริ่มต้น (hidden) และจะแสดงตรงกลางจอด้านบน -->
    <div id="donation-alert" class="hidden absolute top-16 left-1/2 transform -translate-x-1/2 w-full max-w-lg z-50">
        <div class="bg-gradient-to-r from-blue-600 to-purple-600 rounded-3xl p-6 text-center shadow-2xl glow-effect text-white border-4 border-white">
            <h2 class="text-3xl font-bold mb-2 uppercase tracking-wide">🎉 มีคนใจดีโดเนท! 🎉</h2>
            <div class="text-5xl font-extrabold text-yellow-300 my-4 drop-shadow-md">
                <span id="alert-amount">0</span> <span class="text-3xl">บาท</span>
            </div>
            <p class="text-2xl font-semibold mb-2">ขอบคุณคุณ <span id="alert-name" class="text-pink-300">ชื่อผู้โดเนท</span></p>
            <p id="alert-message" class="text-lg text-gray-100 italic bg-black/20 p-3 rounded-xl">"ข้อความโดเนท..."</p>
        </div>
    </div>

    <!-- ============================== -->
    <!-- Scripts & Logic                -->
    <!-- ============================== -->
    <script>
        // ==========================================
        // CONFIGURATION (ตั้งค่าตัวแปรสำคัญตรงนี้)
        // ==========================================
        const CONFIG = {
            PROMPTPAY_ID: "0812345678",      // เบอร์โทรศัพท์ หรือ เลขบัตรประชาชน
            ALERT_DURATION: 8000,            // ระยะเวลาแสดงผล Alert (มิลลิวินาที) - 8 วินาที
            THEME_COLOR_RGB: "59, 130, 246", // สี Glow Effect (RGB)
            
            // ตั้งค่าระบบเสียงอ่าน (TTS)
            TTS_LANG: 'th-TH',               // บังคับภาษาไทย
            TTS_VOLUME: 1.0,                 // ความดัง (0.0 ถึง 1.0)
            TTS_RATE: 1.0,                   // ความเร็ว (0.1 ถึง 10)
            TTS_PITCH: 1.2                   // ระดับเสียง (0 ถึง 2)
        };

        // ตั้งค่าตัวแปร CSS และ QR Code เมื่อโหลดหน้า
        document.documentElement.style.setProperty('--theme-color-rgb', CONFIG.THEME_COLOR_RGB);
        document.getElementById('promptpay-qr').src = `https://promptpay.io/${CONFIG.PROMPTPAY_ID}.png`;

        // ตัวแปรควบคุมคิว (เผื่อมีการโดเนทเข้ามาพร้อมกันหลายคน)
        const donationQueue = [];
        let isAlertPlaying = false;

        // ==========================================
        // CORE FUNCTION: triggerDonation
        // ==========================================
        function triggerDonation(name, amount, message) {
            // นำข้อมูลเข้าคิว
            donationQueue.push({ name, amount, message });
            // ถ้าระบบไม่ได้เล่น Alert อยู่ ให้เริ่มประมวลผลคิว
            if (!isAlertPlaying) {
                processQueue();
            }
        }

        // ==========================================
        // ประมวลผล Alert & TTS
        // ==========================================
        function processQueue() {
            if (donationQueue.length === 0) {
                isAlertPlaying = false;
                return;
            }

            isAlertPlaying = true;
            const donation = donationQueue.shift(); // ดึงข้อมูลคิวแรกออกมา

            const alertBox = document.getElementById('donation-alert');
            
            // อัปเดตข้อมูลบน UI
            document.getElementById('alert-name').innerText = donation.name || "ผู้ไม่ประสงค์ออกนาม";
            document.getElementById('alert-amount').innerText = donation.amount;
            
            const msgElement = document.getElementById('alert-message');
            if (donation.message && donation.message.trim() !== "") {
                msgElement.innerText = `"${donation.message}"`;
                msgElement.style.display = 'block';
            } else {
                msgElement.style.display = 'none';
            }

            // แสดงกล่อง Alert พร้อม Animation
            alertBox.classList.remove('hidden', 'alert-exit');
            alertBox.classList.add('alert-enter');

            // สั่งอ่านเสียง (TTS)
            speakText(`ขอบคุณคุณ ${donation.name} สำหรับโดเนท ${donation.amount} บาท ${donation.message}`);

            // ซ่อนกล่อง Alert เมื่อหมดเวลาที่ตั้งไว้
            setTimeout(() => {
                alertBox.classList.remove('alert-enter');
                alertBox.classList.add('alert-exit');
                
                // รอให้ Animation ออกเสร็จ (0.5วิ) ค่อยรันคิวต่อไป
                setTimeout(() => {
                    alertBox.classList.add('hidden');
                    processQueue();
                }, 500);

            }, CONFIG.ALERT_DURATION);
        }

        // ==========================================
        // ฟังก์ชันอ่านข้อความ (Web Speech API)
        // ==========================================
        function speakText(text) {
            if (!('speechSynthesis' in window)) return;

            // ยกเลิกเสียงที่กำลังอ่านอยู่ (ถ้ามี)
            window.speechSynthesis.cancel();

            const utterance = new SpeechSynthesisUtterance(text);
            utterance.lang = CONFIG.TTS_LANG;
            utterance.volume = CONFIG.TTS_VOLUME;
            utterance.rate = CONFIG.TTS_RATE;
            utterance.pitch = CONFIG.TTS_PITCH;

            window.speechSynthesis.speak(utterance);
        }

        // ==========================================
        // สำหรับนักพัฒนา: จุดเชื่อมต่อ API / Webhook (Real-time)
        // ==========================================
        /*
        ตัวอย่างการเขียน Polling หรือ WebSocket เพื่อรับข้อมูลแบบ Real-time:
        
        // 1. แบบ Fetch Polling (ดึงข้อมูลทุกๆ 3 วินาที)
        setInterval(async () => {
            try {
                const response = await fetch('YOUR_API_ENDPOINT');
                const data = await response.json();
                if (data.isNewDonation) {
                    triggerDonation(data.name, data.amount, data.message);
                }
            } catch (err) { console.error(err); }
        }, 3000);
        */

        // ==========================================
        // สำหรับทดสอบ: กดคีย์บอร์ดปุ่ม 'T' เพื่อจำลองการโดเนท
        // ==========================================
        window.addEventListener('keydown', (e) => {
            if (e.key.toLowerCase() === 't') {
                triggerDonation('Somchai', 500, 'สู้ๆ ครับ เป็นกำลังใจให้');
            }
        });

    </script>
</body>
</html>
