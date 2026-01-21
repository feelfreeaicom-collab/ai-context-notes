# ai-context-notesimport React, { useState, useEffect, useMemo, useRef } from 'react';
import { 
  Zap, 
  Target, 
  Activity, 
  ShieldAlert, 
  Maximize2,
  Sparkles,
  Loader2,
  PlusCircle,
  Dna,
  Layers,
  Fingerprint,
  Volume2,
  ImageIcon,
  SearchCode,
  AlertTriangle,
  RefreshCcw,
  Boxes
} from 'lucide-react';

const apiKey = ""; // API key is provided at runtime

// --- Helper: PCM to WAV for Gemini TTS ---
const pcmToWav = (pcmBase64, sampleRate = 24000) => {
  const binaryString = window.atob(pcmBase64);
  const len = binaryString.length;
  const bytes = new Uint8Array(len);
  for (let i = 0; i < len; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  const buffer = bytes.buffer;
  const view = new DataView(new ArrayBuffer(44 + buffer.byteLength));

  view.setUint32(0, 0x52494646, false); // "RIFF"
  view.setUint32(4, 36 + buffer.byteLength, true); // length
  view.setUint32(8, 0x57415645, false); // "WAVE"
  view.setUint32(12, 0x666d7420, false); // "fmt "
  view.setUint32(16, 16, true); // subchunk1size
  view.setUint16(20, 1, true); // audioformat (PCM)
  view.setUint16(22, 1, true); // numchannels
  view.setUint32(24, sampleRate, true); // samplerate
  view.setUint32(28, sampleRate * 2, true); // byterate
  view.setUint16(32, 2, true); // blockalign
  view.setUint16(34, 16, true); // bitspersample
  view.setUint32(36, 0x64617461, false); // "data"
  view.setUint32(40, buffer.byteLength, true); // subchunk2size

  const pcmView = new Uint8Array(buffer);
  for (let i = 0; i < buffer.byteLength; i++) {
    view.setUint8(44 + i, pcmView[i]);
  }
  const blob = new Blob([view], { type: 'audio/wav' });
  return URL.createObjectURL(blob);
};

// --- 개별 공간 시각화 컴포넌트 ---
const SpatialVisualization = ({ type, resolution, title }) => {
  const dotCount = 120;
  
  const dots = useMemo(() => {
    return Array.from({ length: dotCount }).map((_, i) => ({
      id: i,
      baseX: Math.random() * 100,
      baseY: Math.random() * 100,
      noiseX: (Math.random() - 0.5) * 20,
      noiseY: (Math.random() - 0.5) * 20,
      clusterIdx: Math.floor(Math.random() * 4),
      speed: 0.5 + Math.random()
    }));
  }, []);

  return (
    <div className={`relative w-full h-64 rounded-3xl overflow-hidden border-2 transition-all duration-700 ${
      type === 'hallucination' 
        ? 'border-red-900/20 bg-red-950/5 hover:border-red-500/30' 
        : 'border-blue-900/20 bg-blue-950/5 hover:border-blue-500/30'
    } backdrop-blur-sm group`}>
      <div className="absolute inset-0 opacity-5 bg-[radial-gradient(#ffffff_1px,transparent_1px)] [background-size:20px_20px]"></div>
      
      {dots.map((dot) => {
        let targetX, targetY;
        const progress = resolution / 100;

        if (type === 'hallucination') {
          // 환각: 여러 개의 '거짓 섬'으로 분산 수렴
          const clusters = [[20, 20], [80, 25], [30, 70], [70, 75]];
          const [cx, cy] = clusters[dot.clusterIdx];
          targetX = cx + dot.noiseX * (1 - progress * 0.5);
          targetY = cy + dot.noiseY * (1 - progress * 0.5);
        } else {
          // 수렴: 중앙의 선명한 '로직 루프'로 정렬
          const t = (dot.id / dotCount) * Math.PI * 2;
          targetX = 50 + Math.cos(t) * 35;
          targetY = 50 + Math.sin(t) * 15;
        }

        const currentX = dot.baseX + (targetX - dot.baseX) * progress;
        const currentY = dot.baseY + (targetY - dot.baseY) * progress;

        return (
          <div
            key={dot.id}
            className={`absolute rounded-full transition-all duration-1000 ease-out ${
              type === 'hallucination' ? 'bg-red-500/40' : 'bg-blue-400 shadow-[0_0_12px_rgba(59,130,246,0.6)]'
            }`}
            style={{ 
              left: `${currentX}%`, 
              top: `${currentY}%`,
              width: `${type === 'hallucination' ? 2 : 3}px`,
              height: `${type === 'hallucination' ? 2 : 3}px`,
              opacity: 0.2 + (progress * 0.8)
            }}
          />
        );
      })}

      {/* Overlay Info */}
      <div className="absolute inset-0 p-6 flex flex-col justify-between pointer-events-none">
        <div className="flex justify-between items-start">
          <div className="bg-black/40 backdrop-blur-md px-3 py-1 rounded-full border border-white/10 flex items-center gap-2">
            {type === 'hallucination' ? <AlertTriangle size={12} className="text-red-500" /> : <Target size={12} className="text-blue-500" />}
            <span className="text-[10px] font-bold text-white/70 uppercase tracking-widest">{title}</span>
          </div>
          <div className="font-mono text-[10px] text-white/30">DENSITY_{resolution}%</div>
        </div>
        <div className="h-1 w-full bg-white/5 rounded-full overflow-hidden">
          <div 
            className={`h-full transition-all duration-1000 ${type === 'hallucination' ? 'bg-red-500' : 'bg-blue-500'}`}
            style={{ width: `${resolution}%` }}
          ></div>
        </div>
      </div>
    </div>
  );
};

const App = () => {
  const [hallucinationRes, setHallucinationRes] = useState(25);
  const [groundedRes, setGroundedRes] = useState(25);
  const [isGenerating, setIsGenerating] = useState(false);
  const [loadingAction, setLoadingAction] = useState(null); 
  const [prompt, setPrompt] = useState("");
  const [logs, setLogs] = useState([
    {
      category: "ENGINEERING",
      title: "인지 분해능과 추론 공간",
      quote: "추론 공간이 좁아지고 애매한 토큰 자리는 뭘 의미할까? 진짜 핵심이네... 꼭 세종대왕이 노트북을 던졌다 그 뉴스 이야기네.",
      analysis: "확률 생성의 안개(Hallucination)를 제거하기 위해 엔지니어가 명제의 층위를 고밀도로 분해하여 입력하는 능력.",
      tags: ["High-Resolution", "Token-Density"],
      audioUrl: null,
      imageUrl: null,
      hallucinationReport: null
    }
  ]);

  const fetchGemini = async (url, body, attempt = 0) => {
    try {
      const response = await fetch(url + `?key=${apiKey}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body)
      });
      if (!response.ok) throw new Error('Request Failed');
      return await response.json();
    } catch (error) {
      if (attempt < 4) {
        await new Promise(r => setTimeout(r, Math.pow(2, attempt) * 1000));
        return fetchGemini(url, body, attempt + 1);
      }
      throw error;
    }
  };

  const generateInsight = async () => {
    if (!prompt.trim()) return;
    setIsGenerating(true);
    try {
      const systemPrompt = `You are "Pulse", a context engineer. Output JSON: { "category": string, "title": string, "quote": string, "analysis": string, "tags": string[] }. Korean tone. Professional and deep context.`;
      const data = await fetchGemini(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent`, {
        contents: [{ parts: [{ text: `Topic: ${prompt}` }] }],
        systemInstruction: { parts: [{ text: systemPrompt }] },
        generationConfig: { responseMimeType: "application/json" }
      });
      const newLog = JSON.parse(data.candidates[0].content.parts[0].text);
      setLogs([{ ...newLog, audioUrl: null, imageUrl: null, hallucinationReport: null }, ...logs]);
      setPrompt("");
      setGroundedRes(prev => Math.min(100, prev + 10)); // 수렴성 향상 피드백
    } catch (e) { console.error(e); } finally { setIsGenerating(false); }
  };

  const handleTTS = async (index) => {
    if (logs[index].audioUrl) return;
    setLoadingAction({ index, type: 'tts' });
    try {
      const textToSay = `Say calmly: ${logs[index].analysis}`;
      const data = await fetchGemini(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-tts:generateContent`, {
        contents: [{ parts: [{ text: textToSay }] }],
        generationConfig: { 
          responseModalities: ["AUDIO"],
          speechConfig: { voiceConfig: { prebuiltVoiceConfig: { voiceName: "Kore" } } }
        }
      });
      const audioData = data.candidates[0].content.parts[0].inlineData.data;
      const url = pcmToWav(audioData, 24000);
      const newLogs = [...logs];
      newLogs[index].audioUrl = url;
      setLogs(newLogs);
    } catch (e) { console.error(e); } finally { setLoadingAction(null); }
  };

  const handleImagen = async (index) => {
    if (logs[index].imageUrl) return;
    setLoadingAction({ index, type: 'image' });
    try {
      const imagePrompt = `Hyper-realistic abstract digital visualization of: ${logs[index].title}. Dark cyber-noir aesthetic, deep blue and orange neon highlights.`;
      const data = await fetchGemini(`https://generativelanguage.googleapis.com/v1beta/models/imagen-4.0-generate-001:predict`, {
        instances: { prompt: imagePrompt },
        parameters: { sampleCount: 1 }
      });
      const imageUrl = `data:image/png;base64,${data.predictions[0].bytesBase64Encoded}`;
      const newLogs = [...logs];
      newLogs[index].imageUrl = imageUrl;
      setLogs(newLogs);
    } catch (e) { console.error(e); } finally { setLoadingAction(null); }
  };

  const handleDeepAnalysis = async (index) => {
    if (logs[index].hallucinationReport) return;
    setLoadingAction({ index, type: 'analysis' });
    try {
      const analysisPrompt = `Analyze the reasoning depth and hallucination vulnerability of: "${logs[index].quote}". What context is missing? Korean.`;
      const data = await fetchGemini(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent`, {
        contents: [{ parts: [{ text: analysisPrompt }] }]
      });
      const report = data.candidates[0].content.parts[0].text;
      const newLogs = [...logs];
      newLogs[index].hallucinationReport = report;
      setLogs(newLogs);
    } catch (e) { console.error(e); } finally { setLoadingAction(null); }
  };

  return (
    <div className="min-h-screen bg-[#020408] text-slate-300 p-4 md:p-12 font-sans selection:bg-blue-500/30">
      <div className="max-w-5xl mx-auto">
        {/* Header */}
        <header className="mb-16 border-b border-slate-800 pb-10 flex flex-col md:flex-row justify-between items-start md:items-center gap-8">
          <div className="animate-in fade-in slide-in-from-top-4 duration-700">
            <div className="flex items-center gap-3 mb-3">
              <Boxes className="text-blue-500 w-8 h-8" />
              <h1 className="text-3xl font-black text-white italic tracking-tighter">PULSE: LOGIC_LAB_V4</h1>
            </div>
            <p className="text-slate-500 text-sm font-medium">컨텍스트 엔지니어를 위한 이중 추론 공간 시뮬레이터</p>
          </div>
          
          <div className="flex items-center gap-6 bg-slate-900/40 p-2 rounded-2xl border border-slate-800 backdrop-blur-md">
            <div className="flex items-center gap-3 text-xs font-mono px-4">
              <RefreshCcw className="w-3 h-3 text-blue-400 animate-spin-slow" />
              <span className="text-slate-500">ENGINE_SYNC_ACTIVE</span>
            </div>
          </div>
        </header>

        {/* Dual Space Analysis Section */}
        <section className="mb-24">
          <div className="grid md:grid-cols-2 gap-12">
            {/* Hallucination Space Control */}
            <div className="space-y-6">
              <div className="flex justify-between items-end">
                <h3 className="text-sm font-black text-red-500 uppercase tracking-widest flex items-center gap-2">
                  <AlertTriangle size={16} /> Hallucination Space
                </h3>
                <div className="text-right">
                  <span className="text-[10px] text-slate-500 block">ENTROPY_LVL</span>
                  <span className="text-red-400 font-mono font-bold">{hallucinationRes}%</span>
                </div>
              </div>
              <SpatialVisualization type="hallucination" resolution={hallucinationRes} title="Entropy Distribution" />
              <input 
                type="range" min="0" max="100" value={hallucinationRes} 
                onChange={(e) => setHallucinationRes(parseInt(e.target.value))}
                className="w-full h-1 bg-slate-800 rounded-lg appearance-none accent-red-600"
              />
              <p className="text-xs text-slate-500 leading-relaxed">
                *해상도를 높여도 점들이 흩어지거나 가짜 패턴(False Patterns)으로 모이는 상태입니다. 
                맥락이 고정되지 않았을 때의 '확률적 신기루'를 시각화합니다.
              </p>
            </div>

            {/* Grounded Space Control */}
            <div className="space-y-6">
              <div className="flex justify-between items-end">
                <h3 className="text-sm font-black text-blue-500 uppercase tracking-widest flex items-center gap-2">
                  <Target size={16} /> Grounded Space
                </h3>
                <div className="text-right">
                  <span className="text-[10px] text-slate-500 block">STABILITY_LVL</span>
                  <span className="text-blue-400 font-mono font-bold">{groundedRes}%</span>
                </div>
              </div>
              <SpatialVisualization type="grounded" resolution={groundedRes} title="Logic Convergence" />
              <input 
                type="range" min="0" max="100" value={groundedRes} 
                onChange={(e) => setGroundedRes(parseInt(e.target.value))}
                className="w-full h-1 bg-slate-800 rounded-lg appearance-none accent-blue-600"
              />
              <p className="text-xs text-slate-500 leading-relaxed">
                *해상도가 높아질수록 데이터가 선명한 논리적 궤도로 정렬됩니다. 
                엔지니어링된 제약 조건이 확률 분포를 어떻게 '사실'로 수렴시키는지 보여줍니다.
              </p>
            </div>
          </div>
        </section>

        {/* Insight Generator */}
        <section className="mb-20">
          <div className="bg-gradient-to-r from-blue-600/10 to-transparent p-1 rounded-3xl">
            <div className="bg-[#05070a] p-8 rounded-[22px] border border-white/5 flex flex-col md:flex-row gap-6">
              <div className="flex-1 space-y-4">
                <div className="flex items-center gap-2 text-blue-400 font-bold text-xs uppercase tracking-tighter">
                  <Sparkles size={14} /> Intelligence Interface
                </div>
                <input 
                  type="text" 
                  placeholder="분석하고 싶은 새로운 '변곡점' 주제를 입력하세요..." 
                  className="w-full bg-transparent border-none text-xl font-medium outline-none placeholder:text-slate-700"
                  value={prompt}
                  onChange={(e) => setPrompt(e.target.value)}
                  onKeyDown={(e) => e.key === 'Enter' && generateInsight()}
                />
              </div>
              <button 
                onClick={generateInsight}
                disabled={isGenerating || !prompt.trim()}
                className="bg-white text-black hover:bg-blue-400 px-10 py-4 rounded-2xl font-black text-sm transition-all flex items-center justify-center gap-3 disabled:bg-slate-800 disabled:text-slate-600"
              >
                {isGenerating ? <Loader2 className="animate-spin" size={20} /> : <PlusCircle size={20} />}
                통찰 엔진 가동
              </button>
            </div>
          </div>
        </section>

        {/* Log List */}
        <div className="space-y-20">
          {logs.map((log, i) => (
            <article key={i} className="group relative animate-in fade-in slide-in-from-bottom-8 duration-1000">
              <div className="absolute -left-12 top-0 h-full w-[2px] bg-slate-800 group-hover:bg-blue-900 transition-colors"></div>
              <div className="absolute -left-[51px] top-0 w-3 h-3 rounded-full border-2 border-slate-800 bg-[#020408] group-hover:border-blue-500 transition-colors"></div>
              
              <div className="flex flex-wrap items-center justify-between gap-6 mb-8">
                <div>
                  <div className="flex items-center gap-3 mb-2">
                    <span className="text-[10px] font-black bg-white/5 text-white/40 px-3 py-1 rounded-full border border-white/10 uppercase tracking-widest">
                      {log.category}
                    </span>
                    <h3 className="text-2xl font-black text-white group-hover:text-blue-500 transition-colors">{log.title}</h3>
                  </div>
                  <div className="flex gap-4 mt-2">
                    {log.tags.map(t => <span key={t} className="text-[10px] font-mono text-blue-500/50">#{t}</span>)}
                  </div>
                </div>
                
                <div className="flex items-center gap-3">
                  <button onClick={() => handleTTS(i)} disabled={loadingAction?.index === i} className={`p-3 rounded-2xl border border-white/5 bg-white/5 hover:bg-blue-500/20 transition-all ${log.audioUrl ? 'text-blue-500' : 'text-slate-600'}`}>
                    {loadingAction?.index === i && loadingAction.type === 'tts' ? <Loader2 size={18} className="animate-spin" /> : <Volume2 size={18} />}
                  </button>
                  <button onClick={() => handleImagen(i)} disabled={loadingAction?.index === i} className={`p-3 rounded-2xl border border-white/5 bg-white/5 hover:bg-purple-500/20 transition-all ${log.imageUrl ? 'text-purple-500' : 'text-slate-600'}`}>
                    {loadingAction?.index === i && loadingAction.type === 'image' ? <Loader2 size={18} className="animate-spin" /> : <ImageIcon size={18} />}
                  </button>
                  <button onClick={() => handleDeepAnalysis(i)} disabled={loadingAction?.index === i} className={`p-3 rounded-2xl border border-white/5 bg-white/5 hover:bg-orange-500/20 transition-all ${log.hallucinationReport ? 'text-orange-500' : 'text-slate-600'}`}>
                    {loadingAction?.index === i && loadingAction.type === 'analysis' ? <Loader2 size={18} className="animate-spin" /> : <SearchCode size={18} />}
                  </button>
                </div>
              </div>

              <div className="grid lg:grid-cols-[1fr_300px] gap-12 items-start">
                <div className="space-y-8">
                  <div className="bg-slate-900/40 p-10 rounded-[40px] border border-white/5 shadow-2xl relative overflow-hidden group-hover:bg-slate-900/60 transition-all">
                    <p className="text-white italic font-serif leading-relaxed text-xl lg:text-2xl relative z-10">"{log.quote}"</p>
                    <div className="absolute top-0 right-0 p-10 opacity-5 group-hover:opacity-10 transition-opacity">
                      <Fingerprint size={120} className="text-white" />
                    </div>
                  </div>
                  <div className="flex items-start gap-5 px-4">
                    <div className="bg-blue-600 p-2 rounded-xl mt-1">
                      <Target size={16} className="text-white" />
                    </div>
                    <p className="text-lg text-slate-400 leading-relaxed font-light">{log.analysis}</p>
                  </div>
                </div>

                {log.imageUrl && (
                  <div className="rounded-[40px] overflow-hidden border border-white/10 shadow-[0_0_50px_rgba(0,0,0,0.5)] animate-in zoom-in-95 duration-700">
                    <img src={log.imageUrl} alt="AI Representation" className="w-full h-full object-cover transition-transform duration-2000 hover:scale-125" />
                  </div>
                )}
              </div>

              {/* Collapsible Content */}
              <div className="mt-8 space-y-4">
                {log.audioUrl && (
                  <div className="p-6 bg-blue-900/10 border border-blue-500/20 rounded-3xl flex items-center gap-6 animate-in slide-in-from-left-4">
                    <div className="w-12 h-12 bg-blue-600 rounded-full flex items-center justify-center animate-pulse shadow-lg shadow-blue-500/40">
                      <Volume2 className="text-white" size={20} />
                    </div>
                    <audio controls src={log.audioUrl} className="flex-1 accent-blue-500 h-8" />
                  </div>
                )}
                {log.hallucinationReport && (
                  <div className="p-8 bg-orange-950/10 border border-orange-500/20 rounded-[35px] animate-in slide-in-from-right-4">
                    <h4 className="text-orange-500 text-xs font-black mb-4 flex items-center gap-2 tracking-[0.2em] uppercase">
                      <SearchCode size={14} /> Deep Diagnosis Report
                    </h4>
                    <p className="text-slate-400 text-sm leading-relaxed whitespace-pre-wrap font-mono opacity-80 border-l border-orange-500/30 pl-6">
                      {log.hallucinationReport}
                    </p>
                  </div>
                )}
              </div>
            </article>
          ))}
        </div>

        <footer className="mt-48 pb-20 border-t border-white/5 pt-16 flex flex-col md:flex-row justify-between items-center gap-10 opacity-30">
          <div className="flex items-center gap-6 font-mono text-[10px] tracking-[0.3em] text-slate-500 uppercase">
            <div className="flex items-center gap-2">
              <Dna size={14} className="text-blue-500" />
              <span>Logic Convergence: STABLE</span>
            </div>
            <span>PULSE_CONTEXT_ARCHIVE_v4.0</span>
          </div>
          <p className="text-[10px] font-mono uppercase tracking-widest text-right">
            Generated via Gemini Multi-Modal Intelligence Engine
          </p>
        </footer>
      </div>
    </div>
  );
};

export default App;
