import { useState, useEffect, useRef } from "react";
import { Sparkles, Save, Trash2, Loader2, ImageOff, Wand2 } from "lucide-react";

const STYLES = [
  { label: "Vivid", suffix: "vivid colors, dramatic lighting, hyperdetailed" },
  { label: "Dreamy", suffix: "soft dreamy atmosphere, pastel glow, ethereal" },
  { label: "Cinematic", suffix: "cinematic composition, film grain, moody lighting" },
  { label: "Sketch", suffix: "pencil sketch, hand drawn, crosshatching" },
];

export default function Dreamweaver() {
  const [prompt, setPrompt] = useState("");
  const [styleIdx, setStyleIdx] = useState(0);
  const [current, setCurrent] = useState(null); // {url, prompt, seed, ts}
  const [isGenerating, setIsGenerating] = useState(false);
  const [error, setError] = useState(null);
  const [gallery, setGallery] = useState([]);
  const [galleryLoaded, setGalleryLoaded] = useState(false);
  const [savedIds, setSavedIds] = useState(new Set());
  const loadToken = useRef(0);

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get("gallery", false);
        const items = res ? JSON.parse(res.value) : [];
        setGallery(items);
      } catch (e) {
        setGallery([]);
      } finally {
        setGalleryLoaded(true);
      }
    })();
  }, []);

  async function persistGallery(items) {
    try {
      await window.storage.set("gallery", JSON.stringify(items), false);
    } catch (e) {
      console.error("Failed to save gallery", e);
    }
  }

  function buildUrl(text, seed) {
    const full = `${text}, ${STYLES[styleIdx].suffix}`;
    const encoded = encodeURIComponent(full);
    return `https://image.pollinations.ai/prompt/${encoded}?width=768&height=768&seed=${seed}&nologo=true`;
  }

  function generate() {
    const trimmed = prompt.trim();
    if (!trimmed || isGenerating) return;
    setError(null);
    setIsGenerating(true);
    const seed = Math.floor(Math.random() * 1_000_000);
    const url = buildUrl(trimmed, seed);
    const myToken = ++loadToken.current;

    const img = new Image();
    img.onload = () => {
      if (loadToken.current !== myToken) return;
      setCurrent({ url, prompt: trimmed, seed, ts: Date.now() });
      setIsGenerating(false);
    };
    img.onerror = () => {
      if (loadToken.current !== myToken) return;
      setError("Couldn't render that one — try a different prompt.");
      setIsGenerating(false);
    };
    img.src = url;
  }

  function saveCurrent() {
    if (!current) return;
    const id = `${current.seed}-${current.ts}`;
    if (savedIds.has(id)) return;
    const entry = { id, ...current };
    const next = [entry, ...gallery];
    setGallery(next);
    setSavedIds(new Set([...savedIds, id]));
    persistGallery(next);
  }

  function deleteEntry(id) {
    const next = gallery.filter((g) => g.id !== id);
    setGallery(next);
    persistGallery(next);
  }

  function openFromGallery(item) {
    setCurrent(item);
    setPrompt(item.prompt);
    setError(null);
  }

  const currentId = current ? `${current.seed}-${current.ts}` : null;
  const alreadySaved = currentId && savedIds.has(currentId);

  return (
    <div className="min-h-screen w-full bg-[#0D0B1E] text-[#F0EDFF] font-sans relative overflow-x-hidden">
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;700&family=Inter:wght@400;500;600&display=swap');
        .font-display { font-family: 'Space Grotesk', sans-serif; }
        .font-sans { font-family: 'Inter', sans-serif; }
        @keyframes aura-spin {
          0% { transform: rotate(0deg) scale(1); }
          50% { transform: rotate(180deg) scale(1.08); }
          100% { transform: rotate(360deg) scale(1); }
        }
        .aura {
          background: conic-gradient(from 90deg, #8B7FFF, #FF9F6B, #6C63FF, #8B7FFF);
          filter: blur(50px);
          opacity: 0.55;
          animation: aura-spin 12s linear infinite;
        }
        .aura.idle { animation-duration: 30s; opacity: 0.28; }
        @media (prefers-reduced-motion: reduce) {
          .aura { animation: none; }
        }
        .frame-scroll::-webkit-scrollbar { height: 8px; width: 8px; }
        .frame-scroll::-webkit-scrollbar-thumb { background: #2A2650; border-radius: 8px; }
      `}</style>

      {/* Header */}
      <header className="max-w-5xl mx-auto px-6 pt-10 pb-6 flex items-center gap-3">
        <div className="w-9 h-9 rounded-full bg-gradient-to-br from-[#8B7FFF] to-[#FF9F6B] flex items-center justify-center shrink-0">
          <Wand2 size={18} className="text-[#0D0B1E]" />
        </div>
        <div>
          <h1 className="font-display text-xl font-bold tracking-tight">Dreamweaver</h1>
          <p className="text-xs text-[#8B86A8]">describe it, watch it appear</p>
        </div>
      </header>

      {/* Hero / generator */}
      <main className="max-w-5xl mx-auto px-6 pb-8">
        <div className="grid md:grid-cols-2 gap-8 items-start">
          {/* Left: controls */}
          <div className="flex flex-col gap-4">
            <label className="text-sm text-[#8B86A8] font-medium" htmlFor="prompt-box">
              What do you want to see?
            </label>
            <textarea
              id="prompt-box"
              value={prompt}
              onChange={(e) => setPrompt(e.target.value)}
              onKeyDown={(e) => {
                if (e.key === "Enter" && (e.metaKey || e.ctrlKey)) generate();
              }}
              placeholder="a lighthouse made of stained glass, storm at dusk..."
              rows={4}
              className="w-full resize-none rounded-2xl bg-[#171433] border border-[#2A2650] px-4 py-3 text-sm placeholder:text-[#5A5580] focus:outline-none focus:ring-2 focus:ring-[#8B7FFF] focus-visible:ring-2"
            />

            <div className="flex flex-wrap gap-2">
              {STYLES.map((s, i) => (
                <button
                  key={s.label}
                  onClick={() => setStyleIdx(i)}
                  className={`px-3 py-1.5 rounded-full text-xs font-medium border transition-colors ${
                    styleIdx === i
                      ? "bg-[#8B7FFF] border-[#8B7FFF] text-[#0D0B1E]"
                      : "bg-transparent border-[#2A2650] text-[#8B86A8] hover:border-[#8B7FFF] hover:text-[#F0EDFF]"
                  }`}
                >
                  {s.label}
                </button>
              ))}
            </div>

            <button
              onClick={generate}
              disabled={!prompt.trim() || isGenerating}
              className="flex items-center justify-center gap-2 rounded-2xl bg-gradient-to-r from-[#8B7FFF] to-[#FF9F6B] text-[#0D0B1E] font-display font-bold text-sm py-3 disabled:opacity-40 disabled:cursor-not-allowed hover:brightness-110 transition-all"
            >
              {isGenerating ? (
                <>
                  <Loader2 size={16} className="animate-spin" /> weaving...
                </>
              ) : (
                <>
                  <Sparkles size={16} /> Generate
                </>
              )}
            </button>
            {error && <p className="text-xs text-[#FF9F6B]">{error}</p>}
            <p className="text-[11px] text-[#5A5580]">Tip: ⌘/Ctrl + Enter to generate</p>
          </div>

          {/* Right: preview with aura */}
          <div className="relative aspect-square w-full max-w-md mx-auto">
            <div className={`absolute -inset-6 rounded-full aura ${isGenerating ? "" : "idle"}`} />
            <div className="relative w-full h-full rounded-3xl bg-[#171433] border border-[#2A2650] overflow-hidden flex items-center justify-center">
              {current ? (
                <img
                  src={current.url}
                  alt={current.prompt}
                  className="w-full h-full object-cover"
                />
              ) : (
                <div className="flex flex-col items-center gap-2 text-[#5A5580]">
                  <ImageOff size={28} />
                  <span className="text-xs">nothing conjured yet</span>
                </div>
              )}
            </div>
            {current && (
              <button
                onClick={saveCurrent}
                disabled={alreadySaved}
                className="absolute bottom-3 right-3 flex items-center gap-1.5 rounded-full bg-[#0D0B1E]/80 backdrop-blur border border-[#2A2650] px-3 py-1.5 text-xs font-medium disabled:opacity-50 hover:border-[#8B7FFF] transition-colors"
              >
                <Save size={13} /> {alreadySaved ? "Saved" : "Save to gallery"}
              </button>
            )}
          </div>
        </div>
      </main>

      {/* Gallery */}
      <section className="max-w-5xl mx-auto px-6 pb-16">
        <div className="flex items-baseline justify-between mb-4">
          <h2 className="font-display text-sm font-bold tracking-wide text-[#8B86A8] uppercase">
            Your gallery
          </h2>
          <span className="text-xs text-[#5A5580]">{gallery.length} saved</span>
        </div>

        {!galleryLoaded ? (
          <p className="text-sm text-[#5A5580]">loading...</p>
        ) : gallery.length === 0 ? (
          <div className="rounded-2xl border border-dashed border-[#2A2650] py-12 text-center">
            <p className="text-sm text-[#5A5580]">
              Nothing saved yet — generate something above and hit save.
            </p>
          </div>
        ) : (
          <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
            {gallery.map((item) => (
              <div
                key={item.id}
                className="group relative aspect-square rounded-xl overflow-hidden border border-[#2A2650] cursor-pointer"
                onClick={() => openFromGallery(item)}
              >
                <img
                  src={item.url}
                  alt={item.prompt}
                  className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
                />
                <div className="absolute inset-0 bg-gradient-to-t from-black/70 via-transparent to-transparent opacity-0 group-hover:opacity-100 transition-opacity flex flex-col justify-end p-2">
                  <p className="text-[10px] text-white line-clamp-2">{item.prompt}</p>
                </div>
                <button
                  onClick={(e) => {
                    e.stopPropagation();
                    deleteEntry(item.id);
                  }}
                  className="absolute top-2 right-2 w-6 h-6 rounded-full bg-black/60 backdrop-blur flex items-center justify-center opacity-0 group-hover:opacity-100 transition-opacity hover:bg-red-500/80"
                  aria-label="Delete"
                >
                  <Trash2 size={12} />
                </button>
              </div>
            ))}
          </div>
        )}
      </section>
    </div>
  );
}
