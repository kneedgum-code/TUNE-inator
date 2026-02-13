<!DOCTYPE html>
<html lang="nl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>TUNE-inator - Drum Tuner</title>
    <!-- Externe libraries via CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <style>
        body { margin: 0; overflow-x: hidden; }
        .transition-theme { transition: all 0.7s cubic-bezier(0.4, 0, 0.2, 1); }
        input[type=range]::-webkit-slider-thumb {
            -webkit-appearance: none;
            height: 20px;
            width: 20px;
            border-radius: 50%;
            background: currentColor;
            cursor: pointer;
            border: 2px solid white;
        }
    </style>
</head>
<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;

        const translations = {
            nl: {
                tuner: "Tuner", settings: "Instellingen", start: "START", stop: "STOP",
                tare: "FILTER / REF", back: "TERUG", hertz: "HERTZ", deltaHz: "AFWIJKING",
                detecting: "Grondtoon gevonden", waiting: "Sla rustig bij een lug...",
                kitSettings: "Kit Instellingen", numToms: "Aantal Toms", tomSizes: "Tom Maten (Inches)",
                soundStyle: "Klank Karakter", saveGen: "OPSLAAN & TOEPASSEN",
                era: "Tijdvak (Era)", warning: "Sla zachtjes aan bij de lug voor de beste meting.",
                topLug: "Lug Pitch - Slagvel", bottomLug: "Lug Pitch - Reso", 
                resoType: "Resonantie Karakter", resoDesc: "Relatie tussen Slagvel en Reso"
            }
        };

        const eras = ["60's", "70's", "80's", "90's", "00's", "10's", "20's"];

        const freqToNote = (freq) => {
            if (!freq || freq <= 0) return "-";
            const notes = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"];
            const h = 12 * Math.log2(freq / 440) + 69;
            const i = Math.round(h);
            const note = notes[i % 12];
            const octave = Math.floor(i / 12) - 1;
            return `${note}${octave}`;
        };

        function App() {
            const [view, setView] = useState('tuner');
            const [isRunning, setIsRunning] = useState(false);
            const [currentMode, setCurrentMode] = useState('toms');
            const [frequency, setFrequency] = useState(0);
            const [tareFrequency, setTareFrequency] = useState(null);
            const [statusMsg, setStatusMsg] = useState('');
            const [volumeLevel, setVolumeLevel] = useState(0);
            const [selectedTomIdx, setSelectedTomIdx] = useState(0);

            // Standaard instellingen: 4 toms (10, 12, 14, 16) en Rock stijl
            const [kitSettings, setKitSettings] = useState({
                numToms: 4,
                tomSizes: [10, 12, 14, 16],
                snareSize: 14,
                bassSize: 22,
                soundStyle: 'rock',
                eraIndex: 6,
                resoMode: 0
            });

            const audioContextRef = useRef(null);
            const analyserRef = useRef(null);
            const animationRef = useRef(null);
            const lastUpdateRef = useRef(0);

            const t = translations.nl;

            const calculateBasePitch = (size, type) => {
                let base = 180;
                if (type === 'snare') base = 280 - (size - 13) * 15;
                else if (type === 'bass') base = 75 - (size - 18) * 3;
                else base = 210 - (size - 8) * 11;

                const style = kitSettings.soundStyle;
                if (style === 'metal') base *= 0.85; 
                if (style === 'jazz') base *= 1.3;  
                if (style === 'dry') base *= 0.95;  
                return base;
            };

            const getLugTargets = (targetIdx = selectedTomIdx) => {
                let size = 12;
                let type = currentMode;
                if (currentMode === 'snare') size = kitSettings.snareSize;
                else if (currentMode === 'bass') size = kitSettings.bassSize;
                else size = kitSettings.tomSizes[targetIdx];

                const baseLugPitch = calculateBasePitch(size, type);

                let top = baseLugPitch;
                let bottom = baseLugPitch;

                if (kitSettings.resoMode === -1) {
                    top = baseLugPitch * 0.85;
                    bottom = baseLugPitch * 1.15;
                } else if (kitSettings.resoMode === 1) {
                    top = baseLugPitch * 1.15;
                    bottom = baseLugPitch * 0.85;
                } 

                return {
                    top: Math.round(top),
                    bottom: Math.round(bottom),
                    fundamental: Math.round(baseLugPitch * 0.7)
                };
            };

            const lugTargets = getLugTargets();

            const getResoLabel = () => {
                if (kitSettings.resoMode === -1) return "Warm (Top Lager)";
                if (kitSettings.resoMode === 0) return "Lange Reso (Gelijk)";
                return "Attack (Top Hoger)";
            };

            const getTheme = () => {
                const era = eras[kitSettings.eraIndex];
                const style = kitSettings.soundStyle;
                let bg = "bg-slate-950";
                let card = "bg-slate-900 border-slate-800 shadow-2xl";
                let accent = "blue";
                
                if (style === 'metal') accent = "red";
                if (style === 'jazz') accent = "amber";
                if (style === 'dry') accent = "slate";

                if (era === "60's" || era === "70's") {
                    bg = "bg-stone-900 bg-[radial-gradient(#2c241a_1px,transparent_1px)] [background-size:20px_20px]";
                    card = "bg-stone-800/90 border-orange-900/30 shadow-[10px_10px_0px_0px_rgba(124,45,18,0.3)]";
                } else if (era === "80's" || era === "90's") {
                    bg = "bg-indigo-950 bg-[linear-gradient(to_right,#1e1b4b_1px,transparent_1px),linear-gradient(to_bottom,#1e1b4b_1px,transparent_1px)] [background-size:40px_40px]";
                    card = "bg-slate-900/80 border-fuchsia-500/50 shadow-[0_0_20px_rgba(217,70,239,0.2)] backdrop-blur-md";
                }
                return { bg, card, accent };
            };

            const theme = getTheme();

            const initAudio = async () => {
                try {
                    if (!audioContextRef.current) {
                        audioContextRef.current = new (window.AudioContext || window.webkitAudioContext)();
                    }
                    if (audioContextRef.current.state === 'suspended') await audioContextRef.current.resume();
                    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
                    const source = audioContextRef.current.createMediaStreamSource(stream);
                    const analyser = audioContextRef.current.createAnalyser();
                    analyser.fftSize = 32768; 
                    source.connect(analyser);
                    analyserRef.current = analyser;
                    return true;
                } catch (e) { 
                    console.error("Audio error:", e);
                    return false; 
                }
            };

            const toggleTuner = async () => {
                if (!isRunning) {
                    if (await initAudio()) {
                        setIsRunning(true);
                        requestAnimationFrame(processAudio);
                    }
                } else {
                    setIsRunning(false);
                    setVolumeLevel(0);
                    cancelAnimationFrame(animationRef.current);
                }
            };

            const processAudio = () => {
                if (!analyserRef.current || !isRunning) return;
                const bufferLength = analyserRef.current.frequencyBinCount;
                const dataArray = new Float32Array(bufferLength);
                analyserRef.current.getFloatFrequencyData(dataArray);

                let maxVal = -Infinity;
                let sum = 0;
                for (let i = 0; i < bufferLength; i++) {
                    if (dataArray[i] > maxVal) maxVal = dataArray[i];
                    if (dataArray[i] > -100) sum += Math.pow(10, dataArray[i] / 20);
                }
                setVolumeLevel(Math.min((sum / bufferLength) * 1200, 1));

                const now = Date.now();
                if (maxVal > -45 && (now - lastUpdateRef.current > 300)) {
                    const nyquist = audioContextRef.current.sampleRate / 2;
                    const ranges = { bass: [30, 150], snare: [150, 600], toms: [60, 450] };
                    const [minHz, maxHz] = ranges[currentMode];
                    const lowBin = Math.floor((minHz / nyquist) * bufferLength);
                    const highBin = Math.ceil((maxHz / nyquist) * bufferLength);

                    let peakIdx = -1;
                    let peakAmp = -Infinity;
                    for (let i = lowBin; i < highBin; i++) {
                        if (dataArray[i] > peakAmp) {
                            peakAmp = dataArray[i];
                            peakIdx = i;
                        }
                    }

                    if (peakIdx !== -1) {
                        const freq = peakIdx * (audioContextRef.current.sampleRate / (bufferLength * 2));
                        setFrequency(freq);
                        setStatusMsg(t.detecting);
                        lastUpdateRef.current = now;
                    }
                } else if (maxVal < -70) {
                    setStatusMsg(t.waiting);
                }
                animationRef.current = requestAnimationFrame(processAudio);
            };

            return (
                <div className={`min-h-screen transition-theme ${theme.bg} text-white p-4 font-sans`}>
                    <div className="max-w-md mx-auto pb-10">
                        <header className="flex justify-between items-center mb-6">
                            <div className="flex items-center gap-2">
                                <div className={`w-10 h-10 bg-${theme.accent}-600 rounded-xl flex items-center justify-center font-black shadow-lg italic`}>Ti</div>
                                <div>
                                    <h1 className="text-xl font-black uppercase tracking-tighter leading-none">TUNE-inator</h1>
                                    <span className={`text-[10px] font-bold uppercase tracking-[0.2em] text-${theme.accent}-500`}>{eras[kitSettings.eraIndex]} • {kitSettings.soundStyle}</span>
                                </div>
                            </div>
                            <button onClick={() => setView(view === 'tuner' ? 'settings' : 'tuner')} className="p-3 bg-white/5 rounded-2xl hover:bg-white/10 transition-colors">
                                {view === 'tuner' ? '⚙️' : '✕'}
                            </button>
                        </header>

                        {view === 'tuner' ? (
                            <div className="space-y-4 animate-in fade-in duration-500">
                                
                                <div className="grid grid-cols-2 gap-3">
                                    <div className={`bg-white/5 border border-white/10 rounded-3xl p-4 text-center`}>
                                        <div className="text-[9px] font-black text-slate-500 uppercase tracking-widest mb-1">{t.topLug}</div>
                                        <div className={`text-2xl font-black text-${theme.accent}-400 italic leading-none`}>{lugTargets.top} Hz</div>
                                        <div className="text-[10px] font-bold text-slate-400 mt-1 uppercase">{freqToNote(lugTargets.top)}</div>
                                    </div>
                                    <div className={`bg-white/5 border border-white/10 rounded-3xl p-4 text-center`}>
                                        <div className="text-[9px] font-black text-slate-500 uppercase tracking-widest mb-1">{t.bottomLug}</div>
                                        <div className={`text-2xl font-black text-${theme.accent}-400 italic leading-none`}>{lugTargets.bottom} Hz</div>
                                        <div className="text-[10px] font-bold text-slate-400 mt-1 uppercase">{freqToNote(lugTargets.bottom)}</div>
                                    </div>
                                </div>

                                <div className={`transition-theme ${theme.card} rounded-[2.5rem] p-8 text-center relative overflow-hidden shadow-2xl`}>
                                    <div className="absolute inset-0 pointer-events-none transition-all duration-300" 
                                         style={{ background: `radial-gradient(circle at center, rgba(59, 130, 246, ${volumeLevel * 0.15}) 0%, transparent 70%)` }} />
                                    
                                    <div className="relative z-10 py-4">
                                        <div className={`text-8xl font-black tabular-nums tracking-tighter ${tareFrequency ? (Math.abs(frequency - tareFrequency) < 0.5 ? 'text-green-400' : `text-${theme.accent}-400`) : 'text-white'}`}>
                                            {tareFrequency ? (frequency !== 0 ? (frequency - tareFrequency).toFixed(1) : "0.0") : frequency.toFixed(1)}
                                        </div>
                                        <div className="flex flex-col items-center mt-2">
                                            <div className="text-slate-500 font-bold uppercase tracking-[0.3em] leading-none">{tareFrequency ? t.deltaHz : t.hertz}</div>
                                            {!tareFrequency && frequency > 0 && <div className={`text-sm font-black text-${theme.accent}-500 mt-1`}>{freqToNote(frequency)}</div>}
                                        </div>
                                    </div>

                                    <div className="flex flex-col items-center gap-2 mb-8 relative z-10">
                                        <div className="h-1 w-24 bg-white/5 rounded-full overflow-hidden">
                                            <div className={`h-full bg-${theme.accent}-500 transition-all duration-75`} style={{ width: `${volumeLevel * 100}%` }} />
                                        </div>
                                        <div className="text-[10px] font-black uppercase tracking-widest text-slate-500">{statusMsg}</div>
                                    </div>

                                    <div className="grid grid-cols-3 gap-2 mb-4 relative z-10">
                                        {['bass', 'snare', 'toms'].map(m => (
                                            <button key={m} onClick={() => { setCurrentMode(m); setTareFrequency(null); }} className={`py-3 rounded-2xl text-[10px] font-black uppercase tracking-widest transition-all ${currentMode === m ? `bg-${theme.accent}-600 shadow-lg` : 'bg-white/5 text-slate-500'}`}>{m}</button>
                                        ))}
                                    </div>

                                    {currentMode === 'toms' && (
                                        <div className="flex justify-center flex-wrap gap-3 mb-6 animate-in slide-in-from-top-2">
                                            {kitSettings.tomSizes.map((s, idx) => {
                                                const targets = getLugTargets(idx);
                                                return (
                                                    <button key={idx} onClick={() => setSelectedTomIdx(idx)} className={`flex flex-col items-center justify-center w-14 h-14 rounded-full border transition-all ${selectedTomIdx === idx ? `bg-${theme.accent}-600 border-${theme.accent}-400 shadow-lg scale-110` : 'bg-white/5 border-white/10 text-slate-500'}`}>
                                                        <span className="text-[11px] font-black">{s}"</span>
                                                        <span className={`text-[8px] font-bold uppercase tracking-tighter ${selectedTomIdx === idx ? 'text-white' : 'text-slate-600'}`}>{freqToNote(targets.top)}</span>
                                                    </button>
                                                );
                                            })}
                                        </div>
                                    )}

                                    <div className="flex gap-3 relative z-10">
                                        <button onClick={toggleTuner} className={`flex-[2] py-5 rounded-3xl font-black text-xl transition-all active:scale-95 ${isRunning ? 'bg-red-500' : 'bg-white text-black'}`}>
                                            {isRunning ? t.stop : t.start}
                                        </button>
                                        <button disabled={!isRunning} onClick={() => setTareFrequency(tareFrequency ? null : frequency)} className="flex-1 py-5 rounded-3xl font-black text-xs bg-white/5 text-slate-400">
                                            {tareFrequency ? t.back : t.tare}
                                        </button>
                                    </div>
                                </div>
                                <div className="bg-amber-500/10 border border-amber-500/20 p-4 rounded-3xl text-[10px] text-amber-500 font-bold text-center uppercase tracking-wider leading-tight">
                                    {t.warning}
                                </div>
                            </div>
                        ) : (
                            <div className={`animate-in slide-in-from-bottom-4 duration-500 ${theme.card} p-8 rounded-[2.5rem] space-y-8`}>
                                <h2 className="text-xl font-black uppercase tracking-tight">{t.kitSettings}</h2>
                                
                                <div className="p-5 bg-white/5 rounded-3xl border border-white/10">
                                    <div className="flex justify-between items-end mb-4">
                                        <div>
                                            <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest block">{t.resoType}</label>
                                            <span className="text-[8px] text-slate-600 uppercase tracking-tighter">{t.resoDesc}</span>
                                        </div>
                                        <span className={`text-sm font-black text-${theme.accent}-500 italic uppercase tracking-tighter`}>{getResoLabel()}</span>
                                    </div>
                                    <input type="range" min="-1" max="1" step="1" value={kitSettings.resoMode} onChange={(e) => setKitSettings({...kitSettings, resoMode: parseInt(e.target.value)})} className={`w-full h-1 bg-white/10 rounded-lg appearance-none cursor-pointer accent-${theme.accent}-500`} />
                                    <div className="flex justify-between mt-2 text-[8px] font-bold text-slate-600 uppercase tracking-widest">
                                        <span className="w-1/3 text-left">Top Lager</span>
                                        <span className="w-1/3 text-center">Gelijk</span>
                                        <span className="w-1/3 text-right">Top Hoger</span>
                                    </div>
                                </div>

                                <div className="grid grid-cols-2 gap-4">
                                    <div>
                                        <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest block mb-2">Bass (")</label>
                                        <input type="number" value={kitSettings.bassSize} onChange={(e) => setKitSettings({...kitSettings, bassSize: parseInt(e.target.value) || 0})} className="w-full bg-white/5 border border-white/10 rounded-xl p-3 text-center font-black" />
                                    </div>
                                    <div>
                                        <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest block mb-2">Snare (")</label>
                                        <input type="number" value={kitSettings.snareSize} onChange={(e) => setKitSettings({...kitSettings, snareSize: parseInt(e.target.value) || 0})} className="w-full bg-white/5 border border-white/10 rounded-xl p-3 text-center font-black" />
                                    </div>
                                </div>

                                <div>
                                    <div className="flex justify-between items-end mb-4">
                                        <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest">{t.numToms}</label>
                                        <span className={`text-2xl font-black text-${theme.accent}-500`}>{kitSettings.numToms}</span>
                                    </div>
                                    <input type="range" min="1" max="5" value={kitSettings.numToms} onChange={(e) => {
                                        const n = parseInt(e.target.value);
                                        const s = [...kitSettings.tomSizes];
                                        if (n > s.length) {
                                            while(s.length < n) {
                                                const lastSize = s[s.length - 1] || 10;
                                                s.push(lastSize + 2);
                                            }
                                        }
                                        else s.length = n;
                                        setKitSettings({...kitSettings, numToms: n, tomSizes: s});
                                    }} className={`w-full h-1 bg-white/10 rounded-lg appearance-none cursor-pointer accent-${theme.accent}-500`} />
                                </div>

                                <div className="grid grid-cols-3 gap-3">
                                    {kitSettings.tomSizes.map((s, i) => (
                                        <div key={i}>
                                            <div className="text-[8px] font-bold text-slate-600 mb-1 uppercase tracking-tighter">Tom {i+1}</div>
                                            <input type="number" value={s} onChange={(e) => {
                                                const ns = [...kitSettings.tomSizes];
                                                ns[i] = parseInt(e.target.value) || 0;
                                                setKitSettings({...kitSettings, tomSizes: ns});
                                            }} className="w-full bg-white/5 border border-white/10 rounded-xl p-3 text-center font-black" />
                                        </div>
                                    ))}
                                </div>

                                <div>
                                    <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest block mb-4">{t.soundStyle}</label>
                                    <div className="grid grid-cols-2 gap-2">
                                        {['dry', 'rock', 'jazz', 'metal'].map(s => (
                                            <button key={s} onClick={() => setKitSettings({...kitSettings, soundStyle: s})} className={`py-4 rounded-2xl text-[10px] font-black uppercase tracking-widest transition-all ${kitSettings.soundStyle === s ? `bg-${theme.accent}-600 shadow-lg` : 'bg-white/5 text-slate-500'}`}>{s}</button>
                                        ))}
                                    </div>
                                </div>

                                <div>
                                    <div className="flex justify-between items-end mb-4">
                                        <label className="text-[10px] font-bold text-slate-500 uppercase tracking-widest">{t.era}</label>
                                        <span className={`text-2xl font-black italic text-${theme.accent}-500`}>{eras[kitSettings.eraIndex]}</span>
                                    </div>
                                    <input type="range" min="0" max="6" value={kitSettings.eraIndex} onChange={(e) => setKitSettings({...kitSettings, eraIndex: parseInt(e.target.value)})} className={`w-full h-1 bg-white/10 rounded-lg appearance-none cursor-pointer accent-${theme.accent}-500`} />
                                </div>

                                <button onClick={() => setView('tuner')} className="w-full py-5 bg-white text-black rounded-3xl font-black uppercase tracking-widest shadow-xl active:scale-95 transition-all">
                                    {t.saveGen}
                                </button>
                            </div>
                        )}
                    </div>
                </div>
            );
        }

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
