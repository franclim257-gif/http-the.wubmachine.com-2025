import React, { useEffect, useRef, useState } from "react";

// Wub Machine 2025 — SPA em React + Web Audio API
// - Upload de áudio (MP3/WAV/OGG/M4A)
// - Presets (Wub, Dubstep, Chill, House)
// - Controles: Volume, Velocidade, Filtro, Distorção, Delay
// - Wobble (LFO) no filtro para o efeito "wub"
// - Visualizador de espectro
// - Gravação/Exportação via MediaRecorder
// Observação: este é um protótipo 100% client-side.

export default function App() {
  const [fileName, setFileName] = useState<string | null>(null);
  const [isReady, setIsReady] = useState(false);
  const [isPlaying, setIsPlaying] = useState(false);
  const [isRecording, setIsRecording] = useState(false);
  const [duration, setDuration] = useState(0);
  const [currentTime, setCurrentTime] = useState(0);

  // Controles
  const [volume, setVolume] = useState(0.9);
  const [speed, setSpeed] = useState(1.0); // 0.5x–2.0x
  const [filterCutoff, setFilterCutoff] = useState(120); // Hz a 10kHz
  const [filterQ, setFilterQ] = useState(10);
  const [distortion, setDistortion] = useState(0); // 0–100
  const [delayTime, setDelayTime] = useState(0.22); // s
  const [delayFeedback, setDelayFeedback] = useState(0.35);
  const [wobbleDepth, setWobbleDepth] = useState(0.5); // 0–1
  const [wobbleRate, setWobbleRate] = useState(2.0); // Hz

  // Web Audio refs
  const audioCtxRef = useRef<AudioContext | null>(null);
  const sourceRef = useRef<AudioBufferSourceNode | null>(null);
  const bufferRef = useRef<AudioBuffer | null>(null);
  const gainRef = useRef<GainNode | null>(null);
  const filterRef = useRef<BiquadFilterNode | null>(null);
  const waveShaperRef = useRef<WaveShaperNode | null>(null);
  const delayRef = useRef<DelayNode | null>(null);
  const feedbackGainRef = useRef<GainNode | null>(null);
  const analyserRef = useRef<AnalyserNode | null>(null);
  const lfoOscRef = useRef<OscillatorNode | null>(null);
  const lfoGainRef = useRef<GainNode | null>(null);
  const mediaDestRef = useRef<MediaStreamAudioDestinationNode | null>(null);
  const recorderRef = useRef<MediaRecorder | null>(null);
  const dataChunksRef = useRef<Blob[]>([]);

  const canvasRef = useRef<HTMLCanvasElement | null>(null);
  const rafRef = useRef<number | null>(null);

  // Inicializa grafo de áudio
  const ensureAudio = () => {
    if (!audioCtxRef.current) {
      const ctx = new (window.AudioContext || (window as any).webkitAudioContext)();
      audioCtxRef.current = ctx;

      const srcFilter = ctx.createBiquadFilter();
      srcFilter.type = "lowpass";
      srcFilter.frequency.value = filterCutoff;
      srcFilter.Q.value = filterQ;
      filterRef.current = srcFilter;

      const shaper = ctx.createWaveShaper();
      shaper.curve = makeDistortionCurve(distortion);
      waveShaperRef.current = shaper;

      const delay = ctx.createDelay(5.0);
      delay.delayTime.value = delayTime;
      delayRef.current = delay;

      const feedback = ctx.createGain();
      feedback.gain.value = delayFeedback;
      feedbackGainRef.current = feedback;

      const analyser = ctx.createAnalyser();
      analyser.fftSize = 2048;
      analyserRef.current = analyser;

      const gain = ctx.createGain();
      gain.gain.value = volume;
      gainRef.current = gain;

      // LFO para wobble
      const lfo = ctx.createOscillator();
      lfo.type = "sine";
      lfo.frequency.value = wobbleRate;
      lfoOscRef.current = lfo;

      const lfoGain = ctx.createGain();
      lfoGain.gain.value = wobbleDepth * 2000; // faixa em Hz
      lfoGainRef.current = lfoGain;

      // Destino para gravação
      const mediaDest = ctx.createMediaStreamDestination();
      mediaDestRef.current = mediaDest;

      // Conectar grafo
      // source -> filter -> waveshaper -> delay -> feedback -> analyser -> gain -> destination
      //                                                  ^-----------------------|
      filterRef.current.connect(waveShaperRef.current);
      waveShaperRef.current.connect(delayRef.current);
      delayRef.current.connect(analyserRef.current);
      analyserRef.current.connect(gainRef.current);
      gainRef.current.connect(ctx.destination);
      gainRef.current.connect(mediaDest.stream ? mediaDest : gainRef.current);

      // loop de feedback do delay
      delayRef.current.connect(feedbackGainRef.current);
      feedbackGainRef.current.connect(delayRef.current);

      // LFO modula a freq do filtro
      lfoOscRef.current.connect(lfoGainRef.current);
      lfoGainRef.current.connect(filterRef.current.frequency);
      lfoOscRef.current.start();

      startVisualizer();
    }
  };

  const startVisualizer = () => {
    const canvas = canvasRef.current;
    const analyser = analyserRef.current;
    if (!canvas || !analyser) return;
    const ctx2d = canvas.getContext("2d");
    if (!ctx2d) return;

    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);

    const draw = () => {
      if (!analyser || !canvas || !ctx2d) return;
      analyser.getByteFrequencyData(dataArray);
      ctx2d.clearRect(0, 0, canvas.width, canvas.height);

      const barWidth = (canvas.width / bufferLength) * 1.5;
      let x = 0;
      for (let i = 0; i < bufferLength; i++) {
        const value = dataArray[i];
        const barHeight = (value / 255) * canvas.height;
        ctx2d.fillStyle = `rgba(59,130,246,0.85)`; // azul Tailwind 500
        ctx2d.fillRect(x, canvas.height - barHeight, barWidth, barHeight);
        x += barWidth + 1;
      }
      rafRef.current = requestAnimationFrame(draw);
    };

    if (rafRef.current) cancelAnimationFrame(rafRef.current);
    draw();
  };

  const stopPlayback = () => {
    setIsPlaying(false);
    if (sourceRef.current) {
      try { sourceRef.current.stop(); } catch {}
      sourceRef.current.disconnect();
      sourceRef.current = null;
    }
  };

  const loadFile = async (file: File) => {
    ensureAudio();
    const ctx = audioCtxRef.current!;
    const arrayBuf = await file.arrayBuffer();
    const audioBuf = await ctx.decodeAudioData(arrayBuf);
    bufferRef.current = audioBuf;
    setFileName(file.name);
    setIsReady(true);
    setDuration(audioBuf.duration);
    setCurrentTime(0);
  };

  const createSource = () => {
    const ctx = audioCtxRef.current!;
    const src = ctx.createBufferSource();
    src.buffer = bufferRef.current!;
    src.playbackRate.value = speed;
    sourceRef.current = src;
    src.connect(filterRef.current!);
  };

  const play = () => {
    if (!bufferRef.current) return;
    ensureAudio();
    stopPlayback();
    createSource();
    const ctx = audioCtxRef.current!;
    sourceRef.current!.start();
    setIsPlaying(true);

    const startedAt = ctx.currentTime;
    const tick = () => {
      if (!isPlaying) return;
      const t = (ctx.currentTime - startedAt) * speed;
      setCurrentTime(Math.min(t, duration));
      if (t < duration) requestAnimationFrame(tick);
      else setIsPlaying(false);
    };
    requestAnimationFrame(tick);
  };

  const pause = () => {
    stopPlayback();
  };

  // Utilidades
  const makeDistortionCurve = (amount: number) => {
    const k = amount * 10; // leve a forte
    const n = 44100;
    const curve = new Float32Array(n);
    const deg = Math.PI / 180;
    for (let i = 0; i < n; ++i) {
      const x = (i * 2) / n - 1;
      curve[i] = ((3 + k) * x * 20 * deg) / (Math.PI + k * Math.abs(x));
    }
    return curve;
  };

  // Handlers de UI
  useEffect(() => {
    if (!audioCtxRef.current) return;
    if (filterRef.current) {
      filterRef.current.frequency.value = Math.max(20, Math.min(20000, filterCutoff));
      filterRef.current.Q.value = filterQ;
    }
  }, [filterCutoff, filterQ]);

  useEffect(() => {
    if (!audioCtxRef.current) return;
    if (gainRef.current) gainRef.current.gain.value = volume;
  }, [volume]);

  useEffect(() => {
    if (sourceRef.current) sourceRef.current.playbackRate.value = speed;
  }, [speed]);

  useEffect(() => {
    if (!audioCtxRef.current) return;
    if (waveShaperRef.current) waveShaperRef.current.curve = makeDistortionCurve(distortion);
  }, [distortion]);

  useEffect(() => {
    if (!audioCtxRef.current) return;
    if (delayRef.current) delayRef.current.delayTime.value = delayTime;
    if (feedbackGainRef.current) feedbackGainRef.current.gain.value = delayFeedback;
  }, [delayTime, delayFeedback]);

  useEffect(() => {
    if (!audioCtxRef.current) return;
    if (lfoOscRef.current) lfoOscRef.current.frequency.value = wobbleRate;
    if (lfoGainRef.current) lfoGainRef.current.gain.value = wobbleDepth * 2000;
  }, [wobbleRate, wobbleDepth]);

  // Gravação
  const startRecording = () => {
    try {
      ensureAudio();
      const stream = mediaDestRef.current!.stream;
      const rec = new MediaRecorder(stream);
      recorderRef.current = rec;
      dataChunksRef.current = [];
      rec.ondataavailable = (e) => dataChunksRef.current.push(e.data);
      rec.onstop = () => {
        const blob = new Blob(dataChunksRef.current, { type: "audio/webm" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = `${(fileName || "wub")}-remix-${Date.now()}.webm`;
        a.click();
        URL.revokeObjectURL(url);
      };
      rec.start();
      setIsRecording(true);
    } catch (e) {
      alert("Falha ao iniciar gravação: " + (e as Error).message);
    }
  };

  const stopRecording = () => {
    try {
      recorderRef.current?.stop();
      setIsRecording(false);
    } catch (e) {
      alert("Falha ao parar gravação");
    }
  };

  // Presets
  const applyPreset = (p: "wub" | "dubstep" | "chill" | "house") => {
    if (p === "wub") {
      setSpeed(0.95);
      setFilterCutoff(250);
      setFilterQ(12);
      setDistortion(25);
      setDelayTime(0.28);
      setDelayFeedback(0.42);
      setWobbleDepth(0.9);
      setWobbleRate(2.0);
    } else if (p === "dubstep") {
      setSpeed(0.9);
      setFilterCutoff(180);
      setFilterQ(18);
      setDistortion(60);
      setDelayTime(0.32);
      setDelayFeedback(0.55);
      setWobbleDepth(1.0);
      setWobbleRate(1.2);
    } else if (p === "chill") {
      setSpeed(0.85);
      setFilterCutoff(1200);
      setFilterQ(4);
      setDistortion(5);
      setDelayTime(0.24);
      setDelayFeedback(0.25);
      setWobbleDepth(0.25);
      setWobbleRate(0.7);
    } else if (p === "house") {
      setSpeed(1.12);
      setFilterCutoff(2200);
      setFilterQ(6);
      setDistortion(12);
      setDelayTime(0.18);
      setDelayFeedback(0.2);
      setWobbleDepth(0.35);
      setWobbleRate(3.5);
    }
  };

  // Layout helpers
  const Stat = ({ label, value }: { label: string; value: string }) => (
    <div className="px-3 py-2 rounded-2xl bg-slate-800/60 shadow-inner shadow-black/20">
      <div className="text-xs text-slate-400">{label}</div>
      <div className="font-semibold text-slate-100">{value}</div>
    </div>
  );

  const secondsToHMS = (s: number) => {
    const m = Math.floor(s / 60);
    const r = Math.floor(s % 60);
    return `${m}:${String(r).padStart(2, "0")}`;
  };

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-950 via-slate-900 to-slate-950 text-slate-100">
      <div className="max-w-6xl mx-auto px-4 py-8">
        {/* Header */}
        <header className="flex items-center justify-between gap-4">
          <div className="flex items-center gap-3">
            <div className="h-10 w-10 rounded-2xl bg-blue-500/20 grid place-items-center shadow-lg shadow-blue-500/20">
              <span className="text-xl">🎛️</span>
            </div>
            <div>
              <h1 className="text-2xl md:text-3xl font-extrabold tracking-tight">
                Wub Machine <span className="text-blue-400">2025</span>
              </h1>
              <p className="text-sm text-slate-400 -mt-0.5">Remixe músicas no seu navegador • Nova versão</p>
            </div>
          </div>
          <div className="flex items-center gap-2">
            <button
              onClick={() => applyPreset("wub")}
              className="px-3 py-2 rounded-2xl bg-blue-600 hover:bg-blue-500 transition shadow-md"
            >Preset Wub</button>
            <button
              onClick={() => applyPreset("dubstep")}
              className="px-3 py-2 rounded-2xl bg-fuchsia-600 hover:bg-fuchsia-500 transition shadow-md"
            >Dubstep</button>
            <button
              onClick={() => applyPreset("chill")}
              className="px-3 py-2 rounded-2xl bg-emerald-600 hover:bg-emerald-500 transition shadow-md"
            >Chill</button>
            <button
              onClick={() => applyPreset("house")}
              className="px-3 py-2 rounded-2xl bg-amber-600 hover:bg-amber-500 transition shadow-md"
            >House</button>
          </div>
        </header>

        {/* Upload */}
        <section className="mt-8 grid md:grid-cols-3 gap-6">
          <div className="md:col-span-2">
            <label htmlFor="file" className="block">
              <div className="border-2 border-dashed border-slate-700 rounded-3xl p-6 md:p-10 text-center hover:border-blue-500 transition cursor-pointer bg-slate-900/40">
                <div className="text-5xl mb-2">⬆️</div>
                <div className="font-semibold">Solte um arquivo de áudio aqui ou clique para selecionar</div>
                <div className="text-sm text-slate-400 mt-1">MP3, WAV, OGG, M4A</div>
                {fileName && <div className="mt-3 text-blue-300 text-sm">Selecionado: {fileName}</div>}
                <input id="file" type="file" accept="audio/*" className="hidden" onChange={e => {
                  const f = e.target.files?.[0];
                  if (f) loadFile(f);
                }} />
              </div>
            </label>
            <div
              onDragOver={(e) => e.preventDefault()}
              onDrop={(e) => {
                e.preventDefault();
                const f = e.dataTransfer.files?.[0];
                if (f) loadFile(f);
              }}
              className="mt-3 text-xs text-slate-400 text-center"
            >
              Dica: também pode arrastar e soltar o arquivo aqui.
            </div>

            {/* Visualizador */}
            <div className="mt-6 rounded-3xl bg-slate-900/60 p-4 shadow-inner shadow-black/20">
              <div className="flex items-center justify-between">
                <div className="text-sm text-slate-400">Visualizador</div>
                <div className="flex gap-2">
                  <Stat label="Duração" value={duration ? secondsToHMS(duration) : "--:--"} />
                  <Stat label="Tempo" value={secondsToHMS(currentTime)} />
                </div>
              </div>
              <canvas ref={canvasRef} width={900} height={200} className="w-full h-48 mt-3 rounded-2xl bg-black/40" />
            </div>
          </div>

          {/* Controles */}
          <div className="space-y-4">
            <div className="rounded-3xl p-5 bg-slate-900/60 shadow-lg shadow-black/20">
              <div className="flex gap-2 mb-3">
                {!isPlaying ? (
                  <button onClick={play} disabled={!isReady} className="flex-1 px-4 py-3 rounded-2xl bg-blue-600 disabled:bg-slate-700 hover:bg-blue-500 font-semibold">Play</button>
                ) : (
                  <button onClick={pause} className="flex-1 px-4 py-3 rounded-2xl bg-rose-600 hover:bg-rose-500 font-semibold">Pause</button>
                )}
                {!isRecording ? (
                  <button onClick={startRecording} disabled={!isReady} className="px-4 py-3 rounded-2xl bg-emerald-600 disabled:bg-slate-700 hover:bg-emerald-500 font-semibold">Gravar</button>
                ) : (
                  <button onClick={stopRecording} className="px-4 py-3 rounded-2xl bg-amber-600 hover:bg-amber-500 font-semibold">Parar</button>
                )}
              </div>
              <div className="grid grid-cols-1 gap-4">
                <Slider label={`Volume: ${(volume*100).toFixed(0)}%`} min={0} max={1} step={0.01} value={volume} on={setVolume} />
                <Slider label={`Velocidade: ${speed.toFixed(2)}x`} min={0.5} max={2} step={0.01} value={speed} on={setSpeed} />
                <Slider label={`Filtro LPF: ${filterCutoff.toFixed(0)} Hz`} min={60} max={10000} step={1} value={filterCutoff} on={setFilterCutoff} />
                <Slider label={`Resonância (Q): ${filterQ.toFixed(1)}`} min={0.1} max={24} step={0.1} value={filterQ} on={setFilterQ} />
                <Slider label={`Distorção: ${distortion}`} min={0} max={100} step={1} value={distortion} on={setDistortion} />
                <Slider label={`Delay: ${delayTime.toFixed(2)} s`} min={0} max={1} step={0.01} value={delayTime} on={setDelayTime} />
                <Slider label={`Feedback: ${(delayFeedback*100).toFixed(0)}%`} min={0} max={0.95} step={0.01} value={delayFeedback} on={setDelayFeedback} />
                <Slider label={`Wobble Depth`} min={0} max={1.2} step={0.01} value={wobbleDepth} on={setWobbleDepth} />
                <Slider label={`Wobble Rate: ${wobbleRate.toFixed(2)} Hz`} min={0.1} max={10} step={0.1} value={wobbleRate} on={setWobbleRate} />
              </div>
            </div>

            <div className="rounded-3xl p-5 bg-slate-900/60 shadow-inner shadow-black/20">
              <h3 className="font-semibold mb-2">Como funciona</h3>
              <ol className="list-decimal list-inside text-sm text-slate-300 space-y-1">
                <li>Carregue um arquivo de áudio.</li>
                <li>Escolha um preset ou ajuste os controles.</li>
                <li>Clique em Play para ouvir.</li>
                <li>Use Gravar para exportar o remix (WEBM).</li>
              </ol>
              <p className="text-xs text-slate-400 mt-3">Nota: Pitch shift independente de tempo não está incluso neste protótipo. Para isso, precisaríamos de um algoritmo de time-stretch (por ex. WSOLA/PhaseVocoder) rodando em Worklet.</p>
            </div>
          </div>
        </section>

        {/* Rodapé */}
        <footer className="mt-10 text-center text-xs text-slate-500">
          <div>© 2025 Wub Machine — Protótipo não oficial para demonstração.</div>
          <div>Funciona localmente no seu navegador; nenhum áudio é enviado para servidores.</div>
        </footer>
      </div>
    </div>
  );
}

function Slider({ label, min, max, step, value, on }: { label: string; min: number; max: number; step: number; value: number; on: (v: number) => void; }) {
  return (
    <div>
      <div className="text-sm mb-1 text-slate-300">{label}</div>
      <input
        type="range"
        min={min}
        max={max}
        step={step}
        value={value}
        onChange={(e) => on(Number(e.target.value))}
        className="w-full accent-blue-500"
      />
    </div>
  );
}
