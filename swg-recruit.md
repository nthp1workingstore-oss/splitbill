import React, { useState, useEffect, useRef } from 'react';
import { 
  Camera, Upload, ChevronRight, ChevronLeft, CheckCircle, 
  FileText, User, Briefcase, Activity, Shield, AlertTriangle, 
  CreditCard, Phone, Truck, Award, BrainCircuit, ScanFace,
  Sparkles, Volume2, Loader2, MessageSquare
} from 'lucide-react';
import { initializeApp } from "firebase/app";
import { getFirestore, collection, addDoc, serverTimestamp } from "firebase/firestore";

// --- Gemini API Configuration ---
// ระบบจะใส่ Key ให้อัตโนมัติในสภาพแวดล้อมนี้
const apiKey = ""; 

// --- ส่วนตั้งค่า Firebase ---
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID"
};

let db;
try {
  const app = initializeApp(firebaseConfig);
  db = getFirestore(app);
} catch (error) {
  console.log("Firebase not configured yet (Running in Offline Mode)");
}

const App = () => {
  const [step, setStep] = useState(1);
  const [loading, setLoading] = useState(false);
  const [completed, setCompleted] = useState(false);
  const [legalAccepted, setLegalAccepted] = useState(false);
  const [aiScore, setAiScore] = useState(0);
  
  // State สำหรับ Gemini Features
  const [aiSummary, setAiSummary] = useState('');
  const [aiQuestions, setAiQuestions] = useState([]); // New Feature: Interview Questions
  const [isAnalyzing, setIsAnalyzing] = useState(false);
  const [isGeneratingQuestions, setIsGeneratingQuestions] = useState(false); // New Loading State
  const [isSpeaking, setIsSpeaking] = useState(false);
  
  const [formData, setFormData] = useState({
    // 1. ส่วนตัว
    firstName: '', lastName: '', nickname: '', idCard: '', 
    birthDate: '', phone: '', religion: 'พุทธ',
    // 2. กายภาพ & ยูนิฟอร์ม
    height: '', weight: '', bloodType: 'O', 
    shirtSize: 'L', waistSize: '', shoeSize: '',
    congenitalDisease: 'ไม่มี',
    // 3. ทักษะ & ใบอนุญาต
    licenseDriver: 'ไม่มี', licenseSecurity: 'ไม่มี', 
    skillFire: false, skillCPR: false, skillTraffic: false, skillComputer: false,
    // 4. ประวัติ
    militaryStatus: 'ผ่านการเกณฑ์แล้ว',
    experience: 'ไม่เคย', prevCompany: '', prevSalary: '',
    criminalRecord: 'ไม่มี',
    // 5. ฉุกเฉิน & การเงิน
    emergencyName: '', emergencyRel: '', emergencyPhone: '',
    bankName: '', bankAccount: '',
    profileImage: null
  });

  const totalSteps = 6;

  const calculateAge = (birthDate) => {
    if (!birthDate) return 0;
    const birth = new Date(birthDate);
    const now = new Date();
    let age = now.getFullYear() - birth.getFullYear();
    const m = now.getMonth() - birth.getMonth();
    if (m < 0 || (m === 0 && now.getDate() < birth.getDate())) age--;
    return age;
  };

  useEffect(() => {
    let score = 0;
    const age = calculateAge(formData.birthDate);
    const h = parseInt(formData.height) || 0;
    const w = parseInt(formData.weight) || 0;

    if (age >= 20 && age <= 50) score += 20;
    if (h >= 165) score += 15;
    if (w >= 55 && w <= 90) score += 10;
    if (formData.criminalRecord === 'ไม่มี') score += 30;
    if (formData.militaryStatus === 'ผ่านการเกณฑ์แล้ว') score += 10;
    if (formData.experience !== 'ไม่เคย') score += 10;
    if (formData.licenseSecurity === 'มีใบอนุญาตแล้ว') score += 15;
    if (formData.licenseDriver !== 'ไม่มี') score += 5;
    if (formData.skillFire) score += 5;
    if (formData.congenitalDisease !== 'ไม่มี') score -= 20;

    setAiScore(Math.min(100, Math.max(0, score)));
  }, [formData]);

  const handleChange = (e) => {
    const { name, value, type, checked } = e.target;
    setFormData(prev => ({ 
      ...prev, 
      [name]: type === 'checkbox' ? checked : value 
    }));
  };

  const handleImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onloadend = () => {
        setFormData(prev => ({ ...prev, profileImage: reader.result }));
      };
      reader.readAsDataURL(file);
    }
  };

  // --- Gemini Feature 1: Generate AI Summary ---
  const generateAISummary = async () => {
    setIsAnalyzing(true);
    try {
      const prompt = `
        ทำหน้าที่เป็น HR มืออาชีพของบริษัทรักษาความปลอดภัย SWG.
        วิเคราะห์ข้อมูลผู้สมัครงานคนนี้ และเขียน "บทสรุปผู้บริหาร (Executive Summary)" สั้นๆ 3-4 บรรทัด เป็นภาษาไทย
        เพื่อแนะนำผู้สมัครคนนี้ให้กับหัวหน้างาน โดยเน้นจุดแข็งด้านร่างกาย ทักษะ และความน่าเชื่อถือ
        
        ข้อมูลผู้สมัคร:
        ชื่อ: ${formData.firstName} ${formData.lastName}
        อายุ: ${calculateAge(formData.birthDate)}
        รูปร่าง: สูง ${formData.height} ซม. หนัก ${formData.weight} กก.
        ประสบการณ์ รปภ.: ${formData.experience}
        ประวัติอาชญากรรม: ${formData.criminalRecord}
        ใบอนุญาต: ${formData.licenseSecurity}
        ทักษะพิเศษ: ${formData.skillFire ? 'ดับเพลิง,' : ''} ${formData.skillCPR ? 'CPR,' : ''} ${formData.skillTraffic ? 'จราจร' : ''}
      `;

      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }]
          })
        }
      );

      const data = await response.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text || "ไม่สามารถวิเคราะห์ข้อมูลได้ในขณะนี้";
      setAiSummary(text);
    } catch (error) {
      console.error("AI Error:", error);
      setAiSummary("เกิดข้อผิดพลาดในการเชื่อมต่อกับ AI");
    } finally {
      setIsAnalyzing(false);
    }
  };

  // --- Gemini Feature 2: Generate Interview Questions (New!) ---
  const generateInterviewQuestions = async () => {
    setIsGeneratingQuestions(true);
    try {
      const prompt = `
        สร้าง "คำถามสัมภาษณ์งาน รปภ." จำนวน 3 ข้อ สำหรับผู้สมัครคนนี้ โดยพิจารณาจากข้อมูล:
        ประสบการณ์: ${formData.experience}
        ทักษะ: ${formData.skillFire ? 'ดับเพลิง' : ''} ${formData.skillCPR ? 'CPR' : ''}
        
        โจทย์:
        1. ถ้าไม่มีประสบการณ์ ให้ถามเรื่องความพร้อมและการรับมือความกดดัน
        2. ถ้ามีประสบการณ์ ให้ถามเรื่องการแก้ปัญหาเฉพาะหน้าในอดีต
        3. คำถามต้องสุภาพและเป็นภาษาไทย
        4. ตอบกลับมาเป็น JSON Array ของ Strings เท่านั้น เช่น ["คำถามที่ 1", "คำถามที่ 2", "คำถามที่ 3"]
      `;

      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
        {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            generationConfig: {
                responseMimeType: "application/json"
            }
          })
        }
      );

      const data = await response.json();
      const text = data.candidates?.[0]?.content?.parts?.[0]?.text;
      const questions = JSON.parse(text);
      setAiQuestions(questions);
    } catch (error) {
      console.error("AI Questions Error:", error);
      setAiQuestions(["ไม่สามารถสร้างคำถามได้ในขณะนี้", "โปรดลองใหม่อีกครั้ง"]);
    } finally {
      setIsGeneratingQuestions(false);
    }
  };

  // --- Gemini Feature 3: Text-to-Speech ---
  const playTTS = async (text) => {
    if (isSpeaking) return;
    setIsSpeaking(true);
    try {
      const response = await fetch(
        `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent?key=${apiKey}`,
        {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({
            contents: [{ parts: [{ text: text }] }],
            generationConfig: {
              responseModalities: ["AUDIO"],
              speechConfig: {
                voiceConfig: {
                  prebuiltVoiceConfig: { voiceName: "Puck" } // Deep voice suitable for security context
                }
              }
            }
          }),
        }
      );
      
      const data = await response.json();
      const audioContent = data.candidates?.[0]?.content?.parts?.[0]?.inlineData?.data;
      
      if (audioContent) {
        const binaryString = atob(audioContent);
        const bytes = new Uint8Array(binaryString.length);
        for (let i = 0; i < binaryString.length; i++) {
            bytes[i] = binaryString.charCodeAt(i);
        }
        
        const audioContext = new (window.AudioContext || window.webkitAudioContext)();
        
         const audioBuffer = audioContext.createBuffer(1, bytes.length / 2, 24000);
         const channelData = audioBuffer.getChannelData(0);
         const dataView = new DataView(bytes.buffer);
         for (let i = 0; i < bytes.length / 2; i++) {
           channelData[i] = dataView.getInt16(i * 2, true) / 32768;
         }
         
         const source = audioContext.createBufferSource();
         source.buffer = audioBuffer;
         source.connect(audioContext.destination);
         source.start(0);
         source.onended = () => setIsSpeaking(false);
         
      } else {
        setIsSpeaking(false);
      }
    } catch (error) {
      console.error("TTS Error:", error);
      setIsSpeaking(false);
    }
  };

  const nextStep = () => setStep(prev => Math.min(prev + 1, totalSteps));
  const prevStep = () => setStep(prev => Math.max(prev - 1, 1));

  const handleSubmit = async () => {
    if (!legalAccepted) return;
    setLoading(true);
    
    if (db) {
      try {
        await addDoc(collection(db, "applications"), {
          ...formData,
          aiScore,
          aiSummary,
          aiQuestions, // Save questions too
          submittedAt: serverTimestamp(),
          status: 'pending'
        });
      } catch (e) {
        console.error("Error adding document: ", e);
      }
    } else {
        await new Promise(resolve => setTimeout(resolve, 1500));
    }

    setLoading(false);
    setCompleted(true);
  };

  const downloadPDF = () => window.print();

  if (completed) {
    return (
      <div className="min-h-screen bg-slate-900 flex items-center justify-center p-4 font-sans text-slate-200 relative overflow-hidden">
        <div className="absolute inset-0 z-0 bg-cover bg-center opacity-30 blur-sm" style={{ backgroundImage: "url('https://img2.pic.in.th/IMG_5254.png')" }}></div>
        <div className="absolute inset-0 z-0 bg-black/60"></div>

        <div className="bg-slate-800/90 backdrop-blur-md p-8 rounded-2xl shadow-2xl max-w-lg w-full text-center border border-slate-600 animate-fade-in relative z-10">
          <div className="absolute top-4 right-4 bg-slate-900/80 p-2 rounded-lg border border-slate-600">
            <div className="text-[10px] text-slate-400 uppercase tracking-widest">AI Score</div>
            <div className={`text-2xl font-bold ${aiScore > 70 ? 'text-green-400' : aiScore > 50 ? 'text-yellow-400' : 'text-red-400'}`}>
                {aiScore}/100
            </div>
          </div>

          <div className="flex justify-center mb-6">
            <div className="bg-green-500/20 p-4 rounded-full shadow-[0_0_20px_rgba(34,197,94,0.3)]">
              <CheckCircle size={64} className="text-green-500" />
            </div>
          </div>
          <h2 className="text-3xl font-bold text-white mb-2">บันทึกข้อมูลสำเร็จ!</h2>
          <p className="text-slate-400 mb-8">
            ระบบได้รับข้อมูลของคุณแล้ว<br/>
            สถานะ: <span className="text-white font-bold">รอเรียกสัมภาษณ์</span>
          </p>
          
          <div className="space-y-4">
            <button onClick={downloadPDF} className="w-full bg-blue-600 hover:bg-blue-500 text-white font-bold py-3 px-6 rounded-lg transition-all flex items-center justify-center gap-2 shadow-lg">
              <FileText size={20} /> ดาวน์โหลดใบสมัคร (PDF)
            </button>
            <button onClick={() => window.location.reload()} className="w-full bg-slate-700 hover:bg-slate-600 text-slate-300 font-bold py-3 px-6 rounded-lg transition-all">
              กลับสู่หน้าหลัก
            </button>
          </div>

          {/* --- PDF PRINT LAYOUT --- */}
          <div className="hidden print:block text-black text-left mt-0 p-8 bg-white absolute top-0 left-0 w-[210mm] min-h-[297mm] z-[9999]">
            <div className="flex justify-between items-start border-b-2 border-black pb-4 mb-4">
               <div className="flex items-center gap-4">
                 <img src="https://img5.pic.in.th/file/secure-sv1/E0F6BAAE-3462-4E8B-87A5-E0E0AF64E9D5c811f66172cab7bc.png" alt="SWG Logo" className="h-20 w-auto" />
                 <div>
                    <h1 className="text-2xl font-bold uppercase">ใบสมัครงาน</h1>
                    <h2 className="text-lg text-gray-600">Application Form</h2>
                 </div>
               </div>
               <div className="text-right text-sm">
                 <p className="font-bold text-lg">SWG SECURITY GROUP</p>
                 <div className="border border-black px-2 py-1 mt-1 inline-block text-center">
                    <p className="text-[10px] text-gray-500">AI PRE-SCORE</p>
                    <p className="font-bold text-xl">{aiScore}</p>
                 </div>
                 <p className="mt-1">วันที่: {new Date().toLocaleDateString('th-TH')}</p>
               </div>
            </div>

            <div className="flex gap-6 mb-6">
                <div className="w-36 h-44 bg-gray-200 border border-black flex items-center justify-center overflow-hidden shrink-0">
                    {formData.profileImage ? <img src={formData.profileImage} className="w-full h-full object-cover"/> : "รูปถ่าย 1 นิ้ว"}
                </div>
                <div className="flex-1">
                    {aiSummary && (
                        <div className="mb-4 p-3 bg-gray-100 border-l-4 border-black text-sm italic">
                            <strong>✨ AI Highlight Summary:</strong><br/>
                            "{aiSummary}"
                        </div>
                    )}
                    <div className="grid grid-cols-2 gap-x-4 gap-y-2 text-sm">
                        <div className="col-span-2 border-b border-gray-300 pb-1 font-bold bg-gray-100 px-2">1. ข้อมูลส่วนตัว (Personal Info)</div>
                        <p><strong>ชื่อ-สกุล:</strong> {formData.firstName} {formData.lastName}</p>
                        <p><strong>ชื่อเล่น:</strong> {formData.nickname} ({calculateAge(formData.birthDate)} ปี)</p>
                        <p><strong>เลขบัตร P.:</strong> {formData.idCard}</p>
                        <p><strong>เบอร์โทร:</strong> {formData.phone}</p>
                        <p className="col-span-2"><strong>ที่อยู่ปัจจุบัน:</strong> {formData.address}</p>
                    </div>
                </div>
            </div>
            
            <div className="grid grid-cols-2 gap-6 text-sm mb-4">
                <div className="space-y-2">
                    <div className="border-b border-gray-300 pb-1 font-bold bg-gray-100 px-2">2. กายภาพ & ยูนิฟอร์ม</div>
                    <p><strong>ส่วนสูง/น้ำหนัก:</strong> {formData.height} / {formData.weight}</p>
                    <p><strong>ขนาดเสื้อ/รองเท้า:</strong> {formData.shirtSize} / {formData.shoeSize}</p>
                    <p><strong>โรคประจำตัว:</strong> {formData.congenitalDisease}</p>
                </div>
                <div className="space-y-2">
                     <div className="border-b border-gray-300 pb-1 font-bold bg-gray-100 px-2">3. ประวัติ & ทักษะ</div>
                     <p><strong>ใบอนุญาต รปภ.:</strong> {formData.licenseSecurity}</p>
                     <p><strong>ประวัติอาชญากรรม:</strong> {formData.criminalRecord}</p>
                     <p><strong>ประสบการณ์:</strong> {formData.experience}</p>
                </div>
            </div>
            
            {aiQuestions && aiQuestions.length > 0 && (
                <div className="mt-4 border border-black p-3 text-sm">
                    <p className="font-bold border-b border-black inline-block mb-2">คำถามสัมภาษณ์ที่แนะนำ (AI Generated):</p>
                    <ul className="list-disc pl-5">
                        {aiQuestions.map((q, i) => <li key={i}>{q}</li>)}
                    </ul>
                </div>
            )}

            <div className="border border-black p-3 text-xs mb-4 mt-auto">
                <strong>บันทึกข้อตกลง:</strong> ข้าพเจ้ารับรองว่าข้อมูลข้างต้นเป็นความจริงทุกประการ ยินยอมให้ตรวจสอบประวัติอาชญากรรม และยินยอมให้บริษัทฯ จัดเก็บข้อมูลตามนโยบาย PDPA
            </div>
            
            <div className="flex justify-between mt-8 pt-4">
                <div className="text-center w-1/3">
                    <div className="border-b border-dotted border-black w-full mb-2"></div>
                    <p>ผู้สมัคร</p>
                </div>
                <div className="text-center w-1/3">
                    <div className="border-b border-dotted border-black w-full mb-2"></div>
                    <p>เจ้าหน้าที่รับสมัคร</p>
                </div>
            </div>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-950 text-slate-200 font-sans pb-10 relative overflow-x-hidden">
      <div className="fixed inset-0 -z-10 bg-cover bg-center opacity-30 mix-blend-overlay" style={{ backgroundImage: "url('https://img2.pic.in.th/IMG_5254.png')" }}></div>
      <div className="fixed inset-0 -z-20 bg-slate-950"></div>
      
      <header className="bg-slate-900/80 backdrop-blur-md border-b border-slate-700/50 sticky top-0 z-40 shadow-lg">
        <div className="max-w-4xl mx-auto px-4 py-3 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <img src="https://img5.pic.in.th/file/secure-sv1/E0F6BAAE-3462-4E8B-87A5-E0E0AF64E9D5c811f66172cab7bc.png" alt="SWG Logo" className="h-14 w-auto object-contain drop-shadow-2xl" />
            <div className="hidden sm:block">
              <h1 className="font-bold text-white text-lg tracking-wide leading-tight">SWG RECRUIT <span className="text-blue-500">PRO</span></h1>
              <p className="text-[10px] text-slate-400 uppercase tracking-widest">Enterprise Recruitment System</p>
            </div>
          </div>
          <div className="text-xs font-bold text-slate-400 bg-slate-800/80 px-3 py-1 rounded-full border border-slate-700">Step {step}/{totalSteps}</div>
        </div>
        <div className="h-1 w-full bg-slate-800/50">
          <div className="h-full bg-blue-500 transition-all duration-300 ease-out shadow-[0_0_10px_rgba(59,130,246,0.5)]" style={{ width: `${(step / totalSteps) * 100}%` }} />
        </div>
      </header>

      <main className="max-w-3xl mx-auto px-4 py-8 relative z-10">
        <div className="bg-slate-900/90 backdrop-blur-sm rounded-2xl border border-slate-700/50 p-6 shadow-2xl relative">
          
          {step === 1 && (
            <div className="animate-fade-in space-y-6">
              <SectionHeader icon={<User />} title="ข้อมูลส่วนตัว (Personal Info)" />
              <div className="flex flex-col items-center mb-6">
                <div className="relative group w-32 h-40 bg-slate-800 rounded-xl border-2 border-dashed border-slate-600 flex flex-col items-center justify-center overflow-hidden hover:border-blue-500 transition-colors">
                  {formData.profileImage ? (
                    <img src={formData.profileImage} className="w-full h-full object-cover" />
                  ) : (
                    <><Camera size={32} className="text-slate-500 mb-2" /><span className="text-xs text-slate-500">แตะเพื่อถ่ายรูป</span></>
                  )}
                  <input type="file" accept="image/*" onChange={handleImageUpload} className="absolute inset-0 opacity-0 cursor-pointer" />
                </div>
              </div>
              <div className="grid grid-cols-2 gap-4">
                <Input name="firstName" label="ชื่อจริง" value={formData.firstName} onChange={handleChange} />
                <Input name="lastName" label="นามสกุล" value={formData.lastName} onChange={handleChange} />
                <Input name="nickname" label="ชื่อเล่น" value={formData.nickname} onChange={handleChange} />
                <Input name="idCard" label="เลขบัตร ปชช." value={formData.idCard} onChange={handleChange} maxLength={13} />
                <Input name="birthDate" label="วันเกิด" type="date" value={formData.birthDate} onChange={handleChange} />
                <Input name="phone" label="เบอร์โทรศัพท์" type="tel" value={formData.phone} onChange={handleChange} />
              </div>
            </div>
          )}

          {step === 2 && (
             <div className="animate-fade-in space-y-6">
               <SectionHeader icon={<Activity />} title="กายภาพ & ยูนิฟอร์ม" />
               <div className="grid grid-cols-2 gap-4">
                  <Input name="height" label="ส่วนสูง (ซม.)" type="number" value={formData.height} onChange={handleChange} />
                  <Input name="weight" label="น้ำหนัก (กก.)" type="number" value={formData.weight} onChange={handleChange} />
                  <div>
                    <label className="block text-sm text-slate-400 mb-1">กรุ๊ปเลือด</label>
                    <select name="bloodType" value={formData.bloodType} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3">
                        {['A', 'B', 'O', 'AB'].map(t => <option key={t}>{t}</option>)}
                    </select>
                  </div>
                  <Input name="waistSize" label="รอบเอว (นิ้ว)" type="number" value={formData.waistSize} onChange={handleChange} />
               </div>
               <div className="bg-slate-800/50 p-4 rounded-xl border border-slate-700">
                  <h3 className="text-sm text-blue-400 font-bold mb-3 flex items-center gap-2"><Truck size={16}/> สำหรับตัดชุดยูนิฟอร์ม</h3>
                  <div className="grid grid-cols-2 gap-4">
                    <div>
                        <label className="block text-sm text-slate-400 mb-1">ขนาดเสื้อ</label>
                        <select name="shirtSize" value={formData.shirtSize} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3">
                            {['S','M','L','XL','2XL','3XL'].map(s => <option key={s}>{s}</option>)}
                        </select>
                    </div>
                    <Input name="shoeSize" label="เบอร์รองเท้า (EU)" type="number" value={formData.shoeSize} onChange={handleChange} placeholder="เช่น 42" />
                  </div>
               </div>
               <div>
                 <label className="block text-sm text-slate-400 mb-1">โรคประจำตัว</label>
                 <textarea name="congenitalDisease" value={formData.congenitalDisease} onChange={handleChange} rows="2" className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3 resize-none" />
               </div>
             </div>
          )}

          {step === 3 && (
             <div className="animate-fade-in space-y-6">
               <SectionHeader icon={<Award />} title="ทักษะ & ใบอนุญาต (Skills)" />
               <div className="space-y-4">
                  <div>
                    <label className="block text-sm text-slate-400 mb-2">ใบอนุญาต ธภ.7 (ใบอนุญาต รปภ.)</label>
                    <select name="licenseSecurity" value={formData.licenseSecurity} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3 text-white">
                        <option value="ไม่มี">ยังไม่มี</option>
                        <option value="มีใบอนุญาตแล้ว">มีใบอนุญาตแล้ว</option>
                        <option value="กำลังดำเนินการ">กำลังดำเนินการ/รอสอบ</option>
                    </select>
                  </div>
                  <div>
                    <label className="block text-sm text-slate-400 mb-2">ใบขับขี่</label>
                    <select name="licenseDriver" value={formData.licenseDriver} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3 text-white">
                        <option value="ไม่มี">ไม่มี</option>
                        <option value="รถจักรยานยนต์">รถจักรยานยนต์</option>
                        <option value="รถยนต์">รถยนต์</option>
                        <option value="ทั้งรถยนต์และมอเตอร์ไซค์">ขับได้ทั้งคู่</option>
                    </select>
                  </div>
                  <div>
                     <label className="block text-sm text-slate-400 mb-3">ทักษะพิเศษ (ติ๊กถูก)</label>
                     <div className="grid grid-cols-2 gap-3">
                        <Checkbox name="skillFire" label="ดับเพลิงขั้นต้น" checked={formData.skillFire} onChange={handleChange} />
                        <Checkbox name="skillCPR" label="ปฐมพยาบาล/CPR" checked={formData.skillCPR} onChange={handleChange} />
                        <Checkbox name="skillTraffic" label="จัดการจราจร" checked={formData.skillTraffic} onChange={handleChange} />
                        <Checkbox name="skillComputer" label="คอมพิวเตอร์พื้นฐาน" checked={formData.skillComputer} onChange={handleChange} />
                     </div>
                  </div>
               </div>
             </div>
          )}

          {step === 4 && (
            <div className="animate-fade-in space-y-6">
              <SectionHeader icon={<Briefcase />} title="ประวัติการทำงาน" />
              <div className="space-y-4">
                <div>
                    <label className="block text-sm text-slate-400 mb-2">สถานะทางทหาร</label>
                    <select name="militaryStatus" value={formData.militaryStatus} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3">
                        <option>ผ่านการเกณฑ์แล้ว</option><option>ได้รับการยกเว้น (รด.)</option><option>ยังไม่เกณฑ์</option>
                    </select>
                </div>
                <div>
                   <label className="block text-sm text-slate-400 mb-2">ประสบการณ์ รปภ.</label>
                   <div className="grid grid-cols-2 gap-2">
                     {['ไม่เคย', 'เคย (1-3 ปี)', 'เคย (>3 ปี)'].map(opt => (
                       <button key={opt} onClick={() => setFormData(prev => ({...prev, experience: opt}))} className={`p-3 rounded-lg border text-sm transition-all ${formData.experience === opt ? 'bg-blue-600 border-blue-500 text-white' : 'bg-slate-800 border-slate-700 text-slate-400'}`}>{opt}</button>
                     ))}
                   </div>
                </div>
                {formData.experience !== 'ไม่เคย' && (
                    <div className="bg-slate-800/50 p-4 rounded-xl border border-slate-700 animate-fade-in">
                        <div className="grid grid-cols-2 gap-4">
                            <Input name="prevCompany" label="บริษัทล่าสุด" value={formData.prevCompany} onChange={handleChange} />
                            <Input name="prevSalary" label="เงินเดือนล่าสุด" type="number" value={formData.prevSalary} onChange={handleChange} />
                        </div>
                    </div>
                )}
                <div>
                    <label className="block text-sm text-slate-400 mb-2">ประวัติอาชญากรรม</label>
                    <div className="flex gap-4 p-3 bg-slate-800 rounded-lg border border-slate-700">
                        <label className="flex items-center gap-2 cursor-pointer"><input type="radio" name="criminalRecord" value="ไม่มี" checked={formData.criminalRecord === 'ไม่มี'} onChange={handleChange} className="accent-green-500 w-5 h-5" /><span className="text-white">ไม่เคยมี</span></label>
                        <label className="flex items-center gap-2 cursor-pointer"><input type="radio" name="criminalRecord" value="เคยมี" checked={formData.criminalRecord === 'เคยมี'} onChange={handleChange} className="accent-red-500 w-5 h-5" /><span className="text-white">เคยมี</span></label>
                    </div>
                </div>
              </div>
            </div>
          )}

          {step === 5 && (
            <div className="animate-fade-in space-y-6">
                <SectionHeader icon={<CreditCard />} title="ข้อมูลติดต่อฉุกเฉิน & การเงิน" />
                <div className="space-y-4">
                    <h3 className="text-sm text-slate-400 border-b border-slate-700 pb-2">บุคคลติดต่อฉุกเฉิน</h3>
                    <div className="grid grid-cols-2 gap-4">
                        <Input name="emergencyName" label="ชื่อ-สกุล" value={formData.emergencyName} onChange={handleChange} />
                        <Input name="emergencyRel" label="ความสัมพันธ์" value={formData.emergencyRel} onChange={handleChange} placeholder="เช่น บิดา/มารดา" />
                        <Input name="emergencyPhone" label="เบอร์โทรฉุกเฉิน" type="tel" value={formData.emergencyPhone} onChange={handleChange} className="col-span-2" />
                    </div>
                </div>
                <div className="space-y-4 mt-6">
                    <h3 className="text-sm text-slate-400 border-b border-slate-700 pb-2">บัญชีธนาคาร (รับเงินเดือน)</h3>
                    <div className="grid grid-cols-2 gap-4">
                        <div>
                            <label className="block text-sm text-slate-400 mb-1">ธนาคาร</label>
                            <select name="bankName" value={formData.bankName} onChange={handleChange} className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3">
                                <option value="">เลือกธนาคาร...</option>
                                <option>กสิกรไทย</option><option>ไทยพาณิชย์</option><option>กรุงเทพ</option><option>กรุงไทย</option><option>ออมสิน</option>
                            </select>
                        </div>
                        <Input name="bankAccount" label="เลขที่บัญชี" type="number" value={formData.bankAccount} onChange={handleChange} />
                    </div>
                </div>
            </div>
          )}

          {step === 6 && (
             <div className="animate-fade-in space-y-6">
                <div className="text-center mb-6">
                  <h2 className="text-2xl font-bold text-white mb-2">ตรวจสอบและยืนยัน</h2>
                  <p className="text-slate-400 text-sm">ระบบกำลังประเมินความเหมาะสมเบื้องต้น...</p>
                </div>

                {/* AI Score */}
                <div className="bg-slate-800 p-6 rounded-xl border border-slate-700 flex items-center justify-between shadow-inner mb-4">
                    <div className="flex items-center gap-4">
                        <div className="bg-blue-600/20 p-3 rounded-full"><BrainCircuit className="text-blue-400" size={32}/></div>
                        <div>
                            <h3 className="font-bold text-white">AI Candidate Score</h3>
                            <p className="text-xs text-slate-400">คะแนนความเหมาะสม</p>
                        </div>
                    </div>
                    <div className="text-right">
                        <span className={`text-4xl font-bold ${aiScore >= 60 ? 'text-green-400' : 'text-orange-400'}`}>{aiScore}</span>
                        <span className="text-slate-500 text-sm">/100</span>
                    </div>
                </div>

                {/* ✨ Gemini Features Container ✨ */}
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                    {/* Feature 1: Summary */}
                    <div className="bg-slate-800 p-5 rounded-xl border border-slate-700/50 shadow-lg">
                        <div className="flex items-center justify-between mb-3">
                            <div className="flex items-center gap-2 text-purple-400">
                                <Sparkles size={20} />
                                <h3 className="font-bold">AI Summary</h3>
                            </div>
                            {!aiSummary && (
                                <button 
                                    onClick={generateAISummary} 
                                    disabled={isAnalyzing}
                                    className="text-xs bg-purple-600 hover:bg-purple-500 text-white px-3 py-1.5 rounded-full transition-all flex items-center gap-1 disabled:opacity-50"
                                >
                                    {isAnalyzing ? <Loader2 size={12} className="animate-spin"/> : <Sparkles size={12}/>}
                                    {isAnalyzing ? '...' : 'สร้างสรุป'}
                                </button>
                            )}
                        </div>
                        
                        {aiSummary ? (
                            <div className="bg-slate-900/50 p-3 rounded-lg text-sm text-slate-300 italic border border-purple-500/30 animate-fade-in h-32 overflow-y-auto custom-scrollbar">
                                "{aiSummary}"
                            </div>
                        ) : (
                            <p className="text-xs text-slate-500">ให้ AI ช่วยสรุปจุดเด่นของคุณเพื่อนำเสนอหัวหน้างาน</p>
                        )}
                    </div>

                    {/* Feature 2: Interview Prep (NEW) */}
                    <div className="bg-slate-800 p-5 rounded-xl border border-slate-700/50 shadow-lg">
                        <div className="flex items-center justify-between mb-3">
                            <div className="flex items-center gap-2 text-blue-400">
                                <MessageSquare size={20} />
                                <h3 className="font-bold">AI Interview Prep</h3>
                            </div>
                            {aiQuestions.length === 0 && (
                                <button 
                                    onClick={generateInterviewQuestions} 
                                    disabled={isGeneratingQuestions}
                                    className="text-xs bg-blue-600 hover:bg-blue-500 text-white px-3 py-1.5 rounded-full transition-all flex items-center gap-1 disabled:opacity-50"
                                >
                                    {isGeneratingQuestions ? <Loader2 size={12} className="animate-spin"/> : <MessageSquare size={12}/>}
                                    {isGeneratingQuestions ? '...' : 'เก็งข้อสอบ'}
                                </button>
                            )}
                        </div>
                        
                        {aiQuestions.length > 0 ? (
                            <div className="bg-slate-900/50 p-3 rounded-lg text-sm text-slate-300 border border-blue-500/30 animate-fade-in h-32 overflow-y-auto custom-scrollbar">
                                <ul className="list-disc pl-4 space-y-1">
                                    {aiQuestions.map((q, i) => (
                                        <li key={i}>{q}</li>
                                    ))}
                                </ul>
                            </div>
                        ) : (
                            <p className="text-xs text-slate-500">กดปุ่มเพื่อดูแนวทางคำถามสัมภาษณ์ที่เหมาะกับคุณ</p>
                        )}
                    </div>
                </div>

                {/* Legal Box with 🔊 Gemini TTS */}
                <div className="bg-slate-900/80 p-4 rounded-lg border border-slate-600 mt-4 relative">
                    <div className="flex items-center justify-between mb-3">
                         <div className="flex items-center gap-2 text-amber-500">
                            <AlertTriangle size={20} />
                            <h3 className="font-bold">ข้อตกลงและเงื่อนไข</h3>
                        </div>
                        <button 
                            onClick={() => playTTS("ข้อหนึ่ง ข้าพเจ้ารับรองว่าข้อมูลทั้งหมดเป็นความจริง หากเป็นเท็จ ยินยอมให้เลิกจ้างทันที. ข้อสอง ยินยอมให้บริษัท SWG ตรวจสอบประวัติอาชญากรรม. ข้อสาม ยินยอมให้เก็บรวบรวมข้อมูลส่วนบุคคล หรือ PDPA เพื่อใช้ในการจ้างงาน")}
                            disabled={isSpeaking}
                            className={`p-2 rounded-full transition-all ${isSpeaking ? 'bg-amber-500/20 text-amber-500 animate-pulse' : 'bg-slate-800 text-slate-400 hover:text-white'}`}
                        >
                            <Volume2 size={18} />
                        </button>
                    </div>
                    
                    <div className="h-32 overflow-y-auto text-xs text-slate-400 space-y-2 pr-2 border border-slate-700 rounded p-2 bg-slate-950 mb-3 custom-scrollbar">
                        <p>1. ข้าพเจ้ารับรองว่าข้อมูลทั้งหมดเป็นความจริง หากเป็นเท็จ ยินยอมให้เลิกจ้างทันที</p>
                        <p>2. ยินยอมให้บริษัท SWG ตรวจสอบประวัติอาชญากรรมจากสำนักงานตำรวจแห่งชาติ</p>
                        <p>3. ยินยอมให้เก็บรวบรวมข้อมูลส่วนบุคคล (PDPA) เพื่อใช้ในการจ้างงาน</p>
                    </div>
                    <label className="flex items-start gap-3 cursor-pointer group">
                        <input type="checkbox" checked={legalAccepted} onChange={(e) => setLegalAccepted(e.target.checked)} className="mt-1 w-5 h-5 accent-blue-500" />
                        <span className="text-sm text-slate-300 group-hover:text-white transition-colors">ข้าพเจ้ายอมรับเงื่อนไขและข้อตกลงทั้งหมด</span>
                    </label>
                </div>
             </div>
          )}

          <div className="flex gap-4 mt-8 pt-6 border-t border-slate-800">
            {step > 1 && (
              <button onClick={prevStep} className="px-6 py-3 rounded-xl bg-slate-800 hover:bg-slate-700 text-slate-300 transition-colors flex items-center gap-2"><ChevronLeft size={20}/> ย้อนกลับ</button>
            )}
            <div className="flex-1"></div>
            {step < totalSteps ? (
              <button onClick={nextStep} className="px-6 py-3 rounded-xl bg-blue-600 hover:bg-blue-500 text-white font-bold transition-colors shadow-lg shadow-blue-900/50 flex items-center gap-2">ถัดไป <ChevronRight size={20}/></button>
            ) : (
              <button onClick={handleSubmit} disabled={loading || !legalAccepted} className={`px-8 py-3 rounded-xl font-bold transition-all shadow-lg flex items-center gap-2 ${!legalAccepted ? 'bg-slate-700 text-slate-500 cursor-not-allowed' : 'bg-green-600 hover:bg-green-500 text-white shadow-green-900/50'}`}>
                {loading ? 'กำลังส่งข้อมูล...' : <><CheckCircle size={20}/> ยืนยันการสมัคร</>}
              </button>
            )}
          </div>

        </div>
      </main>
    </div>
  );
};

const SectionHeader = ({ icon, title }) => (
  <div className="flex items-center gap-3 mb-6 border-b border-slate-800 pb-4">
    <div className="text-blue-400">{React.cloneElement(icon, { size: 28 })}</div>
    <h2 className="text-2xl font-bold text-white">{title}</h2>
  </div>
);

const Input = ({ label, className, ...props }) => (
  <div className={className}>
    <label className="block text-sm text-slate-400 mb-1">{label}</label>
    <input className="w-full bg-slate-800 border border-slate-700 rounded-lg px-4 py-3 text-white focus:border-blue-500 focus:ring-1 focus:ring-blue-500 transition-all outline-none" {...props} />
  </div>
);

const Checkbox = ({ label, checked, onChange, name }) => (
  <label className={`flex items-center gap-3 p-3 rounded-lg border cursor-pointer transition-all ${checked ? 'bg-blue-600/20 border-blue-500' : 'bg-slate-800 border-slate-700'}`}>
    <input type="checkbox" name={name} checked={checked} onChange={onChange} className="w-5 h-5 accent-blue-500" />
    <span className="text-sm text-white">{label}</span>
  </label>
);

export default App;
