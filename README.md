import React, { useState } from 'react';
import { 
  ArrowRightLeft, 
  Copy, 
  Check, 
  Volume2, 
  VolumeX,
  Sparkles, 
  Globe, 
  CheckCircle2, 
  BookOpen, 
  RefreshCw,
  Sliders,
  Zap,
  Loader2
} from 'lucide-react';

// รายการภาษาที่รองรับ
const LANGUAGES = [
  { code: 'auto', name: 'ตรวจจับภาษาอัตโนมัติ (Auto Detect)', ttscode: 'th' },
  { code: 'th', name: 'ไทย (Thai)', ttscode: 'th' },
  { code: 'en', name: 'อังกฤษ (English)', ttscode: 'en' },
  { code: 'zh', name: 'จีน (Chinese)', ttscode: 'zh-CN' },
  { code: 'ja', name: 'ญี่ปุ่น (Japanese)', ttscode: 'ja' },
  { code: 'ko', name: 'เกาหลี (Korean)', ttscode: 'ko' },
  { code: 'es', name: 'สเปน (Spanish)', ttscode: 'es' },
  { code: 'fr', name: 'ฝรั่งเศส (French)', ttscode: 'fr' },
  { code: 'de', name: 'เยอรมัน (German)', ttscode: 'de' },
  { code: 'ru', name: 'รัสเซีย (Russian)', ttscode: 'ru' },
  { code: 'vi', name: 'เวียดนาม (Vietnamese)', ttscode: 'vi' },
  { code: 'id', name: 'อินโดนีเซีย (Indonesian)', ttscode: 'id' }
];

// สไตล์และโทนเสียงการแปล
const TRANSLATION_TONES = [
  { id: 'standard', name: 'ทั่วไป (Standard)', desc: 'แปลตรงตัวตามธรรมชาติ' },
  { id: 'formal', name: 'ทางการ (Formal/Business)', desc: 'เหมาะสำหรับอีเมลและเอกสารการทำงาน' },
  { id: 'casual', name: 'เป็นกันเอง (Casual)', desc: 'ภาษาพูด สนิทสนม' },
  { id: 'slang', name: 'ภาษาวัยรุ่น/สแลง (Slang)', desc: 'แปลตามกระแสฮิตและบริบทโซเชียล' }
};

export default function App() {
  const [sourceLang, setSourceLang] = useState('auto');
  const [targetLang, setTargetLang] = useState('en');
  const [inputText, setInputText] = useState('');
  const [outputText, setOutputText] = useState('');
  const [tone, setTone] = useState('standard');
  const [isLoading, setIsLoading] = useState(false);
  const [copied, setCopied] = useState(false);
  const [detectedLang, setDetectedLang] = useState('');
  
  // ฟีเจอร์เสริมจาก Gemini
  const [grammarCorrection, setGrammarCorrection] = useState('');
  const [culturalNotes, setCulturalNotes] = useState('');
  const [vocabularyList, setVocabularyList] = useState([]);
  const [activeTab, setActiveTab] = useState('translation');

  // สถานะสำหรับการเล่นเสียง
  const [playingState, setPlayingState] = useState(null); // 'source' | 'target' | null
  const [loadingAudio, setLoadingAudio] = useState(null); // 'source' | 'target' | null
  const [currentAudio, setCurrentAudio] = useState(null);

  // แปลง PCM16 เป็นไฟล์ WAV สำหรับเล่นเสียงผ่าน Gemini TTS
  const pcmToWav = (base64Pcm, sampleRate = 24000) => {
    const binaryStr = atob(base64Pcm);
    const len = binaryStr.length;
    const bytes = new Uint8Array(len);
    for (let i = 0; i < len; i++) {
      bytes[i] = binaryStr.charCodeAt(i);
    }
    
    const wavHeader = new ArrayBuffer(44 + bytes.length);
    const view = new DataView(wavHeader);

    // RIFF header
    view.setUint32(0, 0x52494646, false); // "RIFF"
    view.setUint32(4, 36 + bytes.length, true);
    view.setUint32(8, 0x57415645, false); // "WAVE"
    // FMT chunk
    view.setUint32(12, 0x666d7420, false); // "fmt "
    view.setUint32(16, 16, true); // Subchunk1Size
    view.setUint16(20, 1, true); // AudioFormat (PCM = 1)
    view.setUint16(22, 1, true); // NumChannels (Mono = 1)
    view.setUint32(24, sampleRate, true);
    view.setUint32(28, sampleRate * 2, true); // ByteRate
    view.setUint16(32, 2, true); // BlockAlign
    view.setUint16(34, 16, true); // BitsPerSample
    // DATA chunk
    view.setUint32(36, 0x64617461, false); // "data"
    view.setUint32(40, bytes.length, true);

    const wavBytes = new Uint8Array(wavHeader);
    wavBytes.set(bytes, 44);

    return new Blob([wavBytes], { type: 'audio/wav' });
  };

  // ฟังเสียงด้วย Google Translate Voice หรือ Gemini TTS
  const handleSpeak = async (text, langCode, type) => {
    if (!text || !text.trim()) return;

    // หากกำลังเล่นเสียงอยู่ ให้หยุด
    if (currentAudio) {
      currentAudio.pause();
      setCurrentAudio(null);
      if (playingState === type) {
        setPlayingState(null);
        return;
      }
    }

    setLoadingAudio(type);

    // หา รหัสภาษา สำหรับ Google TTS
    const langObj = LANGUAGES.find(l => l.code === langCode);
    const ttsLang = langObj ? langObj.ttscode : 'en';

    // 1. ลองใช้ Google Translate TTS Direct URL (เสียงจาก Google Translate โดยตรง)
    const googleTranslateTtsUrl = `https://translate.google.com/translate_tts?ie=UTF-8&q=${encodeURIComponent(text)}&tl=${ttsLang}&client=tw-ob`;

    try {
      const audio = new Audio(googleTranslateTtsUrl);
      
      audio.oncanplaythrough = () => {
        setLoadingAudio(null);
        setPlayingState(type);
        setCurrentAudio(audio);
        audio.play().catch(() => {
          // หากโดนบล็อก CORS จาก Google Translate URL ให้ไปใช้ Gemini TTS
          fallbackToGeminiTTS(text, type);
        });
      };

      audio.onerror = () => {
        // หากเกิดข้อผิดพลาด ให้สลับไปใช้ Gemini TTS API
        fallbackToGeminiTTS(text, type);
      };

      audio.onended = () => {
        setPlayingState(null);
        setCurrentAudio(null);
      };

    } catch (err) {
      fallbackToGeminiTTS(text, type);
    }
  };

  // ระบบเล่นเสียงสำรองด้วย Gemini TTS (เพื่อความชัวร์ 100%)
  const fallbackToGeminiTTS = async (text, type) => {
    try {
      const apiKey = "";
      const payload = {
        contents: [{ parts: [{ text: `Say clearly in the requested language: ${text}` }] }],
        generationConfig: {
          responseModalities: ["AUDIO"],
          speechConfig: {
            voiceConfig: {
              prebuiltVoiceConfig: { voiceName: "Kore" }
            }
          }
        },
        model: "gemini-2.5-flash-preview-tts"
      };

      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(payload)
        }
      );

      const data = await response.json();
      const base64Audio = data.candidates?.[0]?.content?.parts?.[0]?.inlineData?.data;

      if (base64Audio) {
        const audioBlob = pcmToWav(base64Audio, 24000);
        const audioUrl = URL.createObjectURL(audioBlob);
        const audio = new Audio(audioUrl);

        setLoadingAudio(null);
        setPlayingState(type);
        setCurrentAudio(audio);

        audio.play();
        audio.onended = () => {
          setPlayingState(null);
          setCurrentAudio(null);
        };
      } else {
        throw new Error("No TTS Audio Data");
      }
    } catch (err) {
      // ตัวเลือกสุดท้าย: Web Speech API ในเบราว์เซอร์
      if ('speechSynthesis' in window) {
        window.speechSynthesis.cancel();
        const utterance = new SpeechSynthesisUtterance(text);
        setLoadingAudio(null);
        setPlayingState(type);
        utterance.onend = () => setPlayingState(null);
        window.speechSynthesis.speak(utterance);
      } else {
        setLoadingAudio(null);
        setPlayingState(null);
      }
    }
  };

  // จัดการการส่งคำขอแปลไปยัง Gemini API
  const handleTranslate = async () => {
    if (!inputText.trim()) {
      setOutputText('');
      setGrammarCorrection('');
      setCulturalNotes('');
      setVocabularyList([]);
      return;
    }

    setIsLoading(true);
    const apiKey = "";

    const selectedTargetLangName = LANGUAGES.find(l => l.code === targetLang)?.name || targetLang;
    const selectedSourceLangName = sourceLang === 'auto' ? 'Auto-detect' : LANGUAGES.find(l => l.code === sourceLang)?.name;

    const systemPrompt = `คุณคือผู้เชี่ยวชาญด้านการแปลภาษาและสถาบันภาษาศาสตร์ชั้นนำ
หน้าที่ของคุณคือแปลข้อความจากผู้ใช้ พร้อมให้คำแนะนำด้านภาษาเพิ่มเติม
ตอบกลับในรูปแบบ JSON ตามโครงสร้างนี้เท่านั้น:
{
  "detectedLanguage": "ภาษาที่ตรวจพบ",
  "translation": "ข้อความที่แปลแล้วอย่างถูกต้องและเป็นธรรมชาติ",
  "grammarCorrection": "ข้อเสนอแนะแก้ไขไวยากรณ์ข้อความต้นฉบับ",
  "culturalNotes": "คำอธิบายบริบททางวัฒนธรรม สแลง หรือเกร็ดความรู้เกี่ยวกับคำแปลนี้",
  "vocabulary": [
    {"word": "คำศัพท์สำคัญ", "meaning": "ความหมาย"}
  ]
}`;

    const userPrompt = `
ข้อความต้นฉบับ: "${inputText}"
ภาษาต้นทาง: ${selectedSourceLangName}
ภาษาปลายทาง: ${selectedTargetLangName}
ระดับความเป็นทางการ/โทน: ${TONE_INSTRUCTIONS[tone]}
`;

    const payload = {
      contents: [{ parts: [{ text: userPrompt }] }],
      systemInstruction: { parts: [{ text: systemPrompt }] },
      generationConfig: {
        responseMimeType: "application/json"
      }
    };

    let retries = 0;
    const maxRetries = 5;
    let delay = 1000;

    while (retries <= maxRetries) {
      try {
        const response = await fetch(
          `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
          {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(payload)
          }
        );

        if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);

        const data = await response.json();
        const jsonText = data.candidates?.[0]?.content?.parts?.[0]?.text;

        if (jsonText) {
          const parsed = JSON.parse(jsonText);
          setOutputText(parsed.translation || '');
          setDetectedLang(parsed.detectedLanguage || '');
          setGrammarCorrection(parsed.grammarCorrection || '');
          setCulturalNotes(parsed.culturalNotes || '');
          setVocabularyList(parsed.vocabulary || []);
        }
        break;
      } catch (err) {
        if (retries === maxRetries) {
          console.error("Translation Failed:", err);
          setOutputText("เกิดข้อผิดพลาดในการแปลภาษา กรุณาลองใหม่อีกครั้ง");
        } else {
          await new Promise(res => setTimeout(res, delay));
          delay *= 2;
          retries++;
        }
      }
    }
    setIsLoading(false);
  };

  // เปลี่ยนภาษาต้นทางกับปลายทาง
  const handleSwapLanguages = () => {
    if (sourceLang === 'auto') return;
    const temp = sourceLang;
    setSourceLang(targetLang);
    setTargetLang(temp);
    setInputText(outputText);
    setOutputText(inputText);
  };

  // คัดลอกข้อความ
  const handleCopy = (text) => {
    if (!text) return;
    const textArea = document.createElement("textarea");
    textArea.value = text;
    document.body.appendChild(textArea);
    textArea.select();
    try {
      document.execCommand('copy');
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch (err) {
      console.error('คัดลอกไม่สำเร็จ', err);
    }
    document.body.removeChild(textArea);
  };

  return (
    <div className="min-h-screen bg-slate-50 text-slate-800 flex flex-col font-sans">
      {/* Header */}
      <header className="bg-white border-b border-slate-200 sticky top-0 z-10 shadow-sm">
        <div className="max-w-6xl mx-auto px-4 py-3 flex items-center justify-between">
          <div className="flex items-center space-x-3">
            <div className="bg-indigo-600 p-2 rounded-xl text-white shadow-md shadow-indigo-100">
              <Sparkles className="w-6 h-6 animate-pulse" />
            </div>
            <div>
              <h1 className="font-bold text-xl text-slate-900 tracking-tight flex items-center gap-2">
                Gemini Translator <span className="bg-indigo-100 text-indigo-700 text-xs px-2 py-0.5 rounded-full font-medium">Pro</span>
              </h1>
              <p className="text-xs text-slate-500">ระบบแปลภาษาพร้อมเสียง Google Translate เสียงแท้</p>
            </div>
          </div>
          <div className="flex items-center space-x-2">
            <span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-emerald-100 text-emerald-800">
              <span className="w-1.5 h-1.5 rounded-full bg-emerald-500 mr-1.5 animate-ping"></span>
              Google Translate Voice Enabled
            </span>
          </div>
        </div>
      </header>

      {/* Main Container */}
      <main className="flex-1 max-w-6xl w-full mx-auto p-4 md:p-6 space-y-6">
        
        {/* Controls Header */}
        <div className="bg-white p-4 rounded-2xl border border-slate-200 shadow-sm space-y-4">
          <div className="flex flex-col md:flex-row items-center justify-between gap-4">
            
            {/* Language Selection */}
            <div className="flex items-center justify-between w-full md:w-auto gap-2 flex-1">
              <select 
                value={sourceLang} 
                onChange={(e) => setSourceLang(e.target.value)}
                className="bg-slate-50 border border-slate-300 text-slate-800 text-sm rounded-xl focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 p-2.5 flex-1 font-medium cursor-pointer transition-all"
              >
                {LANGUAGES.map(lang => (
                  <option key={lang.code} value={lang.code}>{lang.name}</option>
                ))}
              </select>

              <button 
                onClick={handleSwapLanguages}
                disabled={sourceLang === 'auto'}
                title="สลับภาษา"
                className="p-2.5 rounded-xl border border-slate-200 hover:bg-slate-100 disabled:opacity-40 disabled:hover:bg-transparent text-slate-600 transition-colors"
              >
                <ArrowRightLeft className="w-5 h-5" />
              </button>

              <select 
                value={targetLang} 
                onChange={(e) => setTargetLang(e.target.value)}
                className="bg-slate-50 border border-slate-300 text-slate-800 text-sm rounded-xl focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 p-2.5 flex-1 font-medium cursor-pointer transition-all"
              >
                {LANGUAGES.filter(l => l.code !== 'auto').map(lang => (
                  <option key={lang.code} value={lang.code}>{lang.name}</option>
                ))}
              </select>
            </div>

            {/* Tone Selector */}
            <div className="flex items-center gap-2 w-full md:w-auto overflow-x-auto pb-1 md:pb-0">
              <Sliders className="w-4 h-4 text-slate-400 hidden sm:block" />
              <div className="flex bg-slate-100 p-1 rounded-xl w-full md:w-auto">
                {TRANSLATION_TONES.map(t => (
                  <button
                    key={t.id}
                    onClick={() => setTone(t.id)}
                    className={`px-3 py-1.5 text-xs font-medium rounded-lg transition-all whitespace-nowrap ${
                      tone === t.id 
                        ? 'bg-white text-indigo-700 shadow-sm font-semibold' 
                        : 'text-slate-600 hover:text-slate-900'
                    }`}
                  >
                    {t.name.split(' ')[0]}
                  </button>
                ))}
              </div>
            </div>

          </div>
        </div>

        {/* Translation Boxes Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          
          {/* Input Box */}
          <div className="bg-white rounded-2xl border border-slate-200 shadow-sm flex flex-col min-h-[280px] focus-within:border-indigo-500 focus-within:ring-1 focus-within:ring-indigo-500 transition-all">
            <div className="p-3 border-b border-slate-100 flex items-center justify-between text-xs text-slate-500 font-medium">
              <span>
                {sourceLang === 'auto' && detectedLang ? `ตรวจพบ: ${detectedLang}` : 'ข้อความต้นฉบับ'}
              </span>
              <span>{inputText.length} ตัวอักษร</span>
            </div>
            
            <textarea
              value={inputText}
              onChange={(e) => setInputText(e.target.value)}
              placeholder="พิมพ์หรือวางข้อความที่ต้องการแปลที่นี่..."
              className="w-full flex-1 p-4 bg-transparent border-0 focus:ring-0 resize-none text-slate-800 placeholder-slate-400 text-base md:text-lg leading-relaxed outline-none"
            />

            <div className="p-3 border-t border-slate-100 flex items-center justify-between bg-slate-50/50 rounded-b-2xl">
              <div className="flex items-center space-x-1">
                {/* ปุ่มฟังเสียงข้อความต้นฉบับ */}
                <button 
                  onClick={() => handleSpeak(inputText, sourceLang, 'source')}
                  disabled={!inputText.trim() || loadingAudio === 'source'}
                  className={`p-2 rounded-lg text-slate-600 transition-colors flex items-center gap-1 text-xs font-medium ${
                    playingState === 'source' ? 'bg-indigo-100 text-indigo-700' : 'hover:bg-slate-200'
                  } disabled:opacity-30`}
                  title="ฟังเสียง Google Translate"
                >
                  {loadingAudio === 'source' ? (
                    <Loader2 className="w-5 h-5 animate-spin text-indigo-600" />
                  ) : playingState === 'source' ? (
                    <VolumeX className="w-5 h-5 text-indigo-600 animate-pulse" />
                  ) : (
                    <Volume2 className="w-5 h-5" />
                  )}
                  {playingState === 'source' && <span className="text-xs">กำลังเล่น...</span>}
                </button>
              </div>

              <button
                onClick={handleTranslate}
                disabled={isLoading || !inputText.trim()}
                className="bg-indigo-600 hover:bg-indigo-700 text-white font-medium px-5 py-2.5 rounded-xl shadow-md shadow-indigo-100 disabled:opacity-50 disabled:shadow-none transition-all flex items-center gap-2"
              >
                {isLoading ? (
                  <>
                    <RefreshCw className="w-4 h-4 animate-spin" />
                    <span>กำลังแปล...</span>
                  </>
                ) : (
                  <>
                    <Zap className="w-4 h-4 fill-current" />
                    <span>แปลภาษา</span>
                  </>
                )}
              </button>
            </div>
          </div>

          {/* Output Box */}
          <div className="bg-white rounded-2xl border border-slate-200 shadow-sm flex flex-col min-h-[280px] relative">
            <div className="p-3 border-b border-slate-100 flex items-center justify-between text-xs text-slate-500 font-medium">
              <span>ผลลัพธ์การแปล</span>
              {tone !== 'standard' && (
                <span className="text-indigo-600 bg-indigo-50 px-2 py-0.5 rounded-md font-medium">
                  โทน: {TRANSLATION_TONES.find(t => t.id === tone)?.name}
                </span>
              )}
            </div>

            <div className="flex-1 p-4 text-slate-800 text-base md:text-lg leading-relaxed overflow-y-auto">
              {isLoading ? (
                <div className="flex flex-col items-center justify-center h-full text-slate-400 space-y-3 py-8">
                  <RefreshCw className="w-8 h-8 animate-spin text-indigo-500" />
                  <p className="text-sm">Gemini กำลังประมวลผลการแปลอย่างละเอียด...</p>
                </div>
              ) : outputText ? (
                <p className="whitespace-pre-wrap">{outputText}</p>
              ) : (
                <span className="text-slate-300">ผลการแปลจะปรากฏที่นี่...</span>
              )}
            </div>

            <div className="p-3 border-t border-slate-100 flex items-center justify-between bg-slate-50/50 rounded-b-2xl">
              <div className="flex items-center space-x-1">
                {/* ปุ่มฟังเสียงคำแปล */}
                <button 
                  onClick={() => handleSpeak(outputText, targetLang, 'target')}
                  disabled={!outputText.trim() || loadingAudio === 'target'}
                  className={`p-2 rounded-lg text-slate-600 transition-colors flex items-center gap-1 text-xs font-medium ${
                    playingState === 'target' ? 'bg-indigo-100 text-indigo-700' : 'hover:bg-slate-200'
                  } disabled:opacity-30`}
                  title="ฟังเสียง Google Translate"
                >
                  {loadingAudio === 'target' ? (
                    <Loader2 className="w-5 h-5 animate-spin text-indigo-600" />
                  ) : playingState === 'target' ? (
                    <VolumeX className="w-5 h-5 text-indigo-600 animate-pulse" />
                  ) : (
                    <Volume2 className="w-5 h-5" />
                  )}
                  {playingState === 'target' && <span className="text-xs">กำลังเล่น...</span>}
                </button>
              </div>

              <button
                onClick={() => handleCopy(outputText)}
                disabled={!outputText}
                className="p-2 rounded-lg hover:bg-slate-200 text-slate-600 disabled:opacity-30 transition-colors flex items-center gap-1.5 text-xs font-medium"
              >
                {copied ? (
                  <>
                    <Check className="w-4 h-4 text-emerald-600" />
                    <span className="text-emerald-600">คัดลอกแล้ว</span>
                  </>
                ) : (
                  <>
                    <Copy className="w-4 h-4" />
                    <span>คัดลอก</span>
                  </>
                )}
              </button>
            </div>
          </div>

        </div>

        {/* AI Insights & Features Tabs */}
        {(grammarCorrection || culturalNotes || vocabularyList.length > 0) && (
          <div className="bg-white rounded-2xl border border-slate-200 shadow-sm overflow-hidden transition-all">
            {/* Tabs Header */}
            <div className="border-b border-slate-200 bg-slate-50/50 px-4 flex space-x-4">
              <button
                onClick={() => setActiveTab('translation')}
                className={`py-3 px-2 text-sm font-medium border-b-2 transition-all flex items-center gap-2 ${
                  activeTab === 'translation'
                    ? 'border-indigo-600 text-indigo-600'
                    : 'border-transparent text-slate-500 hover:text-slate-700'
                }`}
              >
                <CheckCircle2 className="w-4 h-4" />
                <span>การตรวจไวยากรณ์ (Grammar)</span>
              </button>
              <button
                onClick={() => setActiveTab('culture')}
                className={`py-3 px-2 text-sm font-medium border-b-2 transition-all flex items-center gap-2 ${
                  activeTab === 'culture'
                    ? 'border-indigo-600 text-indigo-600'
                    : 'border-transparent text-slate-500 hover:text-slate-700'
                }`}
              >
                <Globe className="w-4 h-4" />
                <span>บริบท & สแลง (Cultural Insights)</span>
              </button>
              <button
                onClick={() => setActiveTab('vocab')}
                className={`py-3 px-2 text-sm font-medium border-b-2 transition-all flex items-center gap-2 ${
                  activeTab === 'vocab'
                    ? 'border-indigo-600 text-indigo-600'
                    : 'border-transparent text-slate-500 hover:text-slate-700'
                }`}
              >
                <BookOpen className="w-4 h-4" />
                <span>คลังคำศัพท์ (Vocabulary)</span>
              </button>
            </div>

            {/* Tab Contents */}
            <div className="p-5">
              {activeTab === 'translation' && (
                <div className="space-y-2">
                  <h3 className="text-sm font-semibold text-slate-900 flex items-center gap-2">
                    <Sparkles className="w-4 h-4 text-indigo-500" />
                    วิเคราะห์และแก้ไขไวยากรณ์ต้นฉบับ
                  </h3>
                  <p className="text-sm text-slate-600 leading-relaxed bg-indigo-50/50 p-3 rounded-xl border border-indigo-100">
                    {grammarCorrection || 'ไม่พบปัญหาไวยากรณ์ ข้อความต้นฉบับสมบูรณ์แล้ว'}
                  </p>
                </div>
              )}

              {activeTab === 'culture' && (
                <div className="space-y-2">
                  <h3 className="text-sm font-semibold text-slate-900 flex items-center gap-2">
                    <Globe className="w-4 h-4 text-indigo-500" />
                    อธิบายบริบททางวัฒนธรรมและความหมายแฝง
                  </h3>
                  <p className="text-sm text-slate-600 leading-relaxed bg-slate-50 p-3 rounded-xl border border-slate-200">
                    {culturalNotes || 'ไม่มีเกร็ดความรู้ทางวัฒนธรรมพิเศษสำหรับประโยคนี้'}
                  </p>
                </div>
              )}

              {activeTab === 'vocab' && (
                <div className="space-y-3">
                  <h3 className="text-sm font-semibold text-slate-900 flex items-center gap-2">
                    <BookOpen className="w-4 h-4 text-indigo-500" />
                    คำศัพท์น่ารู้จากข้อความนี้
                  </h3>
                  {vocabularyList.length > 0 ? (
                    <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-3">
                      {vocabularyList.map((item, idx) => (
                        <div key={idx} className="p-3 bg-slate-50 rounded-xl border border-slate-200">
                          <span className="font-semibold text-indigo-600 text-sm block">{item.word}</span>
                          <span className="text-xs text-slate-500 mt-1 block">{item.meaning}</span>
                        </div>
                      ))}
                    </div>
                  ) : (
                    <p className="text-xs text-slate-400">ไม่มีคำศัพท์แนะนำเป็นพิเศษ</p>
                  )}
                </div>
              )}
            </div>
          </div>
        )}

      </main>

      {/* Footer */}
      <footer className="border-t border-slate-200 bg-white py-4 text-center text-xs text-slate-400">
        <p>ขับเคลื่อนด้วย Google Gemini API & Google Translate TTS Voice Services</p>
      </footer>
    </div>
  );
}

const TONE_INSTRUCTIONS = {
  standard: "แปลตามธรรมชาติ เป็นทางการระดับปานกลาง ถูกต้องตามไวยากรณ์",
  formal: "ใช้ภาษาทางการ ภาษาเขียน สุภาพ เหมาะสำหรับจดหมายธุรกิจหรือเอกสารวิชาการ",
  casual: "ใช้ภาษาพูด ภาษาเป็นกันเอง เข้าใจง่าย สบายๆ",
  slang: "ใช้คำสแลง ภาษาวัยรุ่น หรือคำฮิตในโซเชียลมีเดียให้เข้ากับบริบท"
};

