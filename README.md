# Web Audio DSP Engine v7

Bu proje, tarayıcı üzerinde çalışan gerçek zamanlı bir **DSP (Digital Signal Processing) motoru** ve **ses kaydı sistemi**dir. Kullanıcılar:
- Mikrofon veya ses dosyası üzerinden ses girişi yapabilir.
- Ses üzerinde gerçek zamanlı efekt zincirleri (DSP) çalıştırabilir.
- Ses kaydı alabilir ve kaydı saklayabilir.
- Görselleştirme ile ses aktivitesini izleyebilir.

---

## Ana Özellikler

1. **Dosya veya Mikrofon Girişi**
   - Sürükle-bırak veya input aracılığıyla ses dosyası yükleme
   - Gerçek zamanlı mikrofon sesi yakalama

2. **DSP Zinciri**
   - Gain, Limiter ve diğer audio node’lar ile ses işleme
   - Çıkış sesini başlatmadan önce sessize alma (gain = 0)

3. **Kayıt Sistemi**
   - `MediaRecorder` ile işlenmiş sesi kaydetme
   - Kayıt tamamlandığında otomatik olarak dosya oluşturma

4. **UI / Feedback**
   - Sistem durumunu görselleştirme (`btnInit` yeşil aktif durum)
   - Log sistemi ile hata ve bilgi mesajlarını gösterme

---

## Kod Açıklamaları

```javascript
<!-- File Drop Area -->
<script>
const fi = $('fileInput'); // Dosya input elementini seçiyoruz

// 1. File Input / Drag & Drop
fi.onchange = handleFile; // Input üzerinden dosya seçimi
document.body.addEventListener('dragover', e => e.preventDefault()); // Sürükle-bırak için default davranışı engelle
document.body.addEventListener('drop', e => {
    e.preventDefault();
    if(e.dataTransfer.files.length) {
        const file = e.dataTransfer.files[0];
        if(file.type.startsWith('audio')) {
            handleFile({ target: { files: [file] }}); // Audio dosyasını işle
        }
    }
});

// Dosya yüklendiğinde çalışacak fonksiyon
function handleFile(e) {
    const file = e.target.files[0];
    if(!file) return;

    const audio = new Audio(URL.createObjectURL(file)); // Geçici URL üzerinden ses dosyasını yükle
    audio.loop = true;
    audio.crossOrigin = "anonymous";

    audio.addEventListener('canplay', async () => {
        await initCore(audio); // DSP zincirine bağla
        audio.play();
        log(`Audio File "${file.name}" loaded and playing.`);
    });
}

// 2. initCore fonksiyonu: DSP motorunu başlatır ve kaynakları bağlar
async function initCore(fileStream = null) {
    if(isInit && !fileStream) {
        if(actx.state === 'suspended') await actx.resume(); // AudioContext askıda ise devam ettir
        log("Audio Engine Resumed.");
        return;
    }

    try {
        if(!actx) actx = new (window.AudioContext || window.webkitAudioContext)({ latencyHint: 'interactive' });

        if(src) src.disconnect(); // Önceki kaynağı ayır

        if(fileStream) {
            src = actx.createMediaElementSource(fileStream); // Dosya üzerinden kaynak
            log("Audio File Source Loaded.");
        } else {
            const deviceId = $('audioSource').value; // Seçilen mikrofon cihazı
            const constraints = { 
                audio: { 
                    deviceId: deviceId ? { exact: deviceId } : undefined,
                    echoCancellation: false, noiseSuppression: false, autoGainControl: false,
                    latency: 0
                } 
            };
            const stream = await navigator.mediaDevices.getUserMedia(constraints);
            src = actx.createMediaStreamSource(stream); // Mikrofon kaynağı
            log("Microphone Source Locked.");
        }

        // --- DSP Zincirini oluştur ---
        // [Gain, Limiter ve diğer efektler burada bağlanacak]

        nodes.outGain = actx.createGain(); // Çıkış gain
        nodes.outGain.gain.value = 0; // Başlangıçta sessiz
        nodes.limiter.connect(nodes.outGain);
        nodes.outGain.connect(actx.destination);

        const recDest = actx.createMediaStreamDestination();
        nodes.limiter.connect(recDest);
        mediaRecorder = new MediaRecorder(recDest.stream);
        mediaRecorder.ondataavailable = e => chunks.push(e.data); // Veri geldiğinde sakla
        mediaRecorder.onstop = saveRecording; // Kayıt bittiğinde kaydet

        isInit = true;
        $('btnInit').classList.add('active-green'); // UI güncelle
        $('btnInit').innerHTML = '<i class="fa-solid fa-signal"></i> SYSTEM ONLINE';
        $('btnRec').disabled = false;

        drawViz(); // Görselleştirme
        log("DSP Matrix V7 Configured. All Systems Green.");
    } catch(e) {
        log("INIT ERROR: " + e.message, "err");
        console.error(e);
    }
}
</script>
