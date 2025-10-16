/*
  My_io — Attractive One-Click AI Video Generator
  Single-file React component (TailwindCSS classes). Copy this to `src/App.jsx` in a Vite React project.

  NOTE: This file includes a friendly, modern UI only. Backend endpoints (/api/generate, /api/status/:jobId)
  should still be implemented server-side (see commented section at the bottom for a minimal server.js and package.json).

  Files to add to your project:
  - src/App.jsx           <- paste this component
  - index.css             <- include Tailwind base imports (if using Tailwind)
  - package.json          <- ensure vite + react deps exist (see commented package.json below)
  - server.js             <- backend example (commented below)

  If you'd like, I can also split this into multiple files and generate a ready-to-upload zip for GitHub.
*/

import React, { useState } from 'react';

export default function App() {
  const [topic, setTopic] = useState('What if humans could breathe underwater?');
  const [ratio, setRatio] = useState('16:9');
  const [lengthSec, setLengthSec] = useState(120);
  const [providers, setProviders] = useState({ sora: true, veo: true, speechify: true });
  const [isGenerating, setIsGenerating] = useState(false);
  const [logs, setLogs] = useState([]);
  const [resultUrl, setResultUrl] = useState(null);
  const [thumbnail, setThumbnail] = useState(null);
  const [caption, setCaption] = useState('');

  const addLog = (text) => setLogs((l) => [...l, `${new Date().toLocaleTimeString()} • ${text}`]);

  const toggleProvider = (k) => setProviders((p) => ({ ...p, [k]: !p[k] }));

  const resetResult = () => {
    setResultUrl(null);
    setThumbnail(null);
    setCaption('');
    setLogs([]);
  };

  const validate = () => {
    if (!topic || topic.trim().length < 3) return 'Enter a short topic (3+ chars).';
    if (lengthSec <= 0 || lengthSec > 600) return 'Length must be between 1 s and 600 s.';
    return null;
  };

  const handleGenerate = async () => {
    const err = validate();
    if (err) {
      addLog('⚠️ ' + err);
      return;
    }
    resetResult();
    setIsGenerating(true);
    addLog('Starting job...');

    try {
      // Send request to backend (you must implement /api/generate)
      addLog('Sending request to server');
      const payload = { topic, ratio, lengthSec, providers, platformHints: ['tiktok', 'youtube'] };
      const resp = await fetch('/api/generate', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(payload) });
      if (!resp.ok) {
        const t = await resp.text();
        addLog('❌ Server error: ' + resp.status + ' ' + t);
        setIsGenerating(false);
        return;
      }
      const { jobId } = await resp.json();
      addLog('Job queued: ' + jobId);

      // Poll status
      let attempts = 0;
      while (attempts < 200) {
        await new Promise((r) => setTimeout(r, 2000));
        attempts++;
        const s = await fetch(`/api/status/${encodeURIComponent(jobId)}`);
        if (!s.ok) {
          addLog('Status fetch error: ' + s.status);
          break;
        }
        const json = await s.json();
        if (json.logs && json.logs.length) json.logs.forEach((x) => addLog(x));
        if (json.status === 'done') {
          addLog('✅ Job finished');
          setResultUrl(json.resultUrl);
          setThumbnail(json.thumbnailUrl || null);
          setCaption(json.caption || '');
          setIsGenerating(false);
          return;
        }
        if (json.status === 'failed') {
          addLog('❌ Job failed: ' + (json.error || 'unknown'));
          setIsGenerating(false);
          return;
        }
        addLog('Polling... ' + attempts);
      }
      addLog('⚠️ Polling timeout');
      setIsGenerating(false);
    } catch (e) {
      console.error(e);
      addLog('❌ Unexpected: ' + e.message);
      setIsGenerating(false);
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-900 via-slate-800 to-gray-900 text-slate-100 p-6">
      <div className="max-w-5xl mx-auto bg-gradient-to-br from-white/5 to-white/3 rounded-2xl shadow-2xl p-6 md:p-10 ring-1 ring-white/5">
        <header className="flex items-center justify-between gap-4 mb-6">
          <div className="flex items-center gap-3">
            <div className="w-12 h-12 rounded-xl bg-gradient-to-tr from-indigo-500 to-pink-500 flex items-center justify-center shadow-md">
              <svg width="28" height="28" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
                <path d="M12 2L15 8H9L12 2Z" fill="white" opacity="0.95" />
                <path d="M4 22L12 14L20 22H4Z" fill="white" opacity="0.9" />
              </svg>
            </div>
            <div>
              <h1 className="text-2xl font-semibold">My_io — One-Click AI Video</h1>
              <p className="text-sm text-slate-300">Generate short AI videos with narration, visuals, captions & thumbnail.</p>
            </div>
          </div>
          <div className="text-sm text-slate-400">Max length: 10 min · Ratio: 9:16 / 16:9</div>
        </header>

        <main className="grid grid-cols-1 md:grid-cols-3 gap-6">
          {/* Left: Controls */}
          <div className="md:col-span-2 p-4 bg-slate-800/60 rounded-xl">
            <label className="block text-slate-300 text-sm mb-2">Topic</label>
            <input value={topic} onChange={(e) => setTopic(e.target.value)} className="w-full p-3 rounded-lg bg-slate-900/60 border border-white/5 placeholder:text-slate-400 text-white/95 mb-4" />

            <div className="flex gap-3 mb-4">
              <div className="flex-1">
                <label className="text-slate-300 text-sm">Aspect ratio</label>
                <select value={ratio} onChange={(e) => setRatio(e.target.value)} className="w-full p-2 rounded-md bg-slate-900/60 border border-white/5 text-white/95">
                  <option>16:9</option>
                  <option>9:16</option>
                  <option>1:1</option>
                </select>
              </div>

              <div className="w-44">
                <label className="text-slate-300 text-sm">Length</label>
                <div className="flex items-center gap-2">
                  <input type="number" min={1} max={600} value={lengthSec} onChange={(e) => setLengthSec(Number(e.target.value))} className="w-full p-2 rounded-md bg-slate-900/60 border border-white/5 text-white/95" />
                </div>
                <div className="text-xs text-slate-400 mt-1">seconds (max 600)</div>
              </div>
            </div>

            <div className="flex gap-3 items-center mb-4">
              <label className="inline-flex items-center gap-2 cursor-pointer">
                <input type="checkbox" checked={providers.sora} onChange={() => toggleProvider('sora')} className="accent-indigo-500" />
                <span className="text-slate-300">Sora (visuals)</span>
              </label>
              <label className="inline-flex items-center gap-2 cursor-pointer">
                <input type="checkbox" checked={providers.veo} onChange={() => toggleProvider('veo')} className="accent-pink-500" />
                <span className="text-slate-300">Veo (composition)</span>
              </label>
              <label className="inline-flex items-center gap-2 cursor-pointer">
                <input type="checkbox" checked={providers.speechify} onChange={() => toggleProvider('speechify')} className="accent-emerald-400" />
                <span className="text-slate-300">Speechify (TTS)</span>
              </label>
            </div>

            <div className="flex gap-3">
              <button onClick={handleGenerate} disabled={isGenerating} className="flex-1 inline-flex items-center justify-center gap-3 px-4 py-3 rounded-lg bg-gradient-to-r from-indigo-600 to-pink-600 hover:from-indigo-500 hover:to-pink-500 shadow-md text-white font-medium">
                {isGenerating ? (
                  <svg className="animate-spin h-5 w-5" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4" fill="none"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v4a4 4 0 00-4 4H4z"></path></svg>
                ) : (
                  <svg width="18" height="18" viewBox="0 0 24 24" fill="none"><path d="M5 3v18l15-9L5 3z" fill="white" /></svg>
                )}
                <span>{isGenerating ? 'Generating...' : 'Generate One-Click Video'}</span>
              </button>

              <button onClick={() => { resetResult(); setTopic(''); }} className="px-4 py-3 rounded-lg border border-white/10 text-slate-300">Clear</button>
            </div>

            <div className="mt-4 p-3 bg-slate-900/40 rounded-lg text-sm text-slate-400">
              <strong>Tip:</strong> This UI sends requests to a backend endpoint `/api/generate` which must assemble the video using your provider APIs. Currently the demo uses placeholders.
            </div>
          </div>

          {/* Right: Preview */}
          <aside className="p-4 rounded-xl bg-gradient-to-t from-white/4 to-transparent flex flex-col gap-4">
            <div className="rounded-lg overflow-hidden bg-gradient-to-br from-slate-800 to-slate-700 p-3 shadow-inner">
              <div className="bg-black/50 rounded-md p-2 flex items-center justify-between">
                <div>
                  <div className="text-xs text-slate-300">Thumbnail preview</div>
                  <div className="text-sm text-white/90">Auto-generated when job completes</div>
                </div>
                <div className="text-xs text-slate-400">Ratio: {ratio}</div>
              </div>
              <div className="mt-3 h-40 bg-gradient-to-t from-slate-700 to-slate-800 rounded-md flex items-center justify-center text-slate-400">
                {thumbnail ? <img src={thumbnail} alt="thumb" className="max-h-full" /> : <div className="text-center">No thumbnail yet</div>}
              </div>
            </div>

            <div className="p-3 rounded-md bg-slate-900/30">
              <div className="flex items-center justify-between">
                <div className="text-xs text-slate-300">Caption (SEO)</div>
                <div className="text-xs text-slate-400">Auto</div>
              </div>
              <div className="mt-2 text-sm text-white/90 min-h-[56px]">{caption || 'No caption yet — generated after the job completes.'}</div>
            </div>

            <div className="p-3 rounded-md bg-slate-900/20 flex-1 overflow-auto">
              <div className="text-xs text-slate-300 mb-2">Generation logs</div>
              <div className="text-sm text-slate-200 space-y-1">
                {logs.length === 0 ? <div className="text-slate-400">Logs will appear here when you run a job.</div> : logs.map((l, i) => <div key={i} className="text-xs bg-black/20 p-2 rounded">{l}</div>)}
              </div>
            </div>

            <div className="flex gap-2 mt-2">
              {resultUrl ? <a href={resultUrl} target="_blank" rel="noreferrer" className="flex-1 text-center py-2 rounded bg-emerald-600 text-white">Open Video</a> : <button disabled className="flex-1 text-center py-2 rounded bg-emerald-600/30 text-white/70">Open Video</button>}
              <button onClick={() => { navigator.clipboard?.writeText(caption || ''); }} className="px-3 py-2 rounded border border-white/6 text-slate-300">Copy Caption</button>
            </div>
          </aside>
        </main>

        <footer className="mt-6 text-center text-xs text-slate-400">Made with ⚡️ · Remember to add backend provider keys & moderation APIs before producing real content.</footer>
      </div>
    </div>
  );
}

/* ----------------------
   Minimal package.json (copy into your repo root)
   ----------------------
{
  "name": "my-io",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "start-server": "node server.js"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "express": "^4.18.2",
    "body-parser": "^1.20.2",
    "cors": "^2.8.5",
    "node-fetch": "^2.6.7"
  },
  "devDependencies": {
    "vite": "^5.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "tailwindcss": "^4.3.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0"
  }
}

/* ----------------------
   Minimal server.js example (paste into server.js in project root)
   Replace placeholders with your real provider calls and API keys on the server side.
   ----------------------
const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const crypto = require('crypto');

const app = express();
app.use(cors());
app.use(bodyParser.json({ limit: '5mb' }));

const jobs = {};
function makeJobId() { return crypto.randomBytes(6).toString('hex'); }

app.post('/api/generate', async (req, res) => {
  const { topic } = req.body;
  if (!topic) return res.status(400).json({ error: 'Missing topic' });
  const jobId = makeJobId();
  jobs[jobId] = { status: 'queued', logs: ['Job queued'] };

  // Demo background job (simulate generation)
  (async () => {
    try {
      jobs[jobId].status = 'running';
      jobs[jobId].logs.push('Generating script (demo)');
      await new Promise(r => setTimeout(r, 1200));
      jobs[jobId].logs.push('Generating narration (demo)');
      await new Promise(r => setTimeout(r, 1200));
      jobs[jobId].logs.push('Rendering video (demo)');
      await new Promise(r => setTimeout(r, 2000));
      jobs[jobId].status = 'done';
      jobs[jobId].resultUrl = 'https://example.com/final_video.mp4';
      jobs[jobId].thumbnailUrl = 'https://example.com/thumbnail.png';
      jobs[jobId].caption = `${topic} — A visual journey. #ai #whatif`;
      jobs[jobId].logs.push('Job complete (demo)');
    } catch (e) {
      jobs[jobId].status = 'failed';
      jobs[jobId].error = e.message;
      jobs[jobId].logs.push('Job failed: ' + e.message);
    }
  })();

  res.json({ jobId });
});

app.get('/api/status/:jobId', (req, res) => {
  const j = jobs[req.params.jobId];
  if (!j) return res.status(404).json({ error: 'Not found' });
  res.json(j);
});

app.listen(3001, () => console.log('Server running on port 3001'));
*/
  const [useVeo, setUseVeo] = useState(true);
  const [useSpeechify, setUseSpeechify] = useState(true);
  const [statusLog, setStatusLog] = useState([]);
  const [isGenerating, setIsGenerating] = useState(false);
  const [resultUrl, setResultUrl] = useState(null);
  const [thumbnailPreview, setThumbnailPreview] = useState(null);
  const [caption, setCaption] = useState("");

  const addLog = (line) => setStatusLog((s) => [...s, `${new Date().toLocaleTimeString()} — ${line}`]);

  const validate = () => {
    if (!topic.trim()) return "Topic is required.";
    if (lengthSec <= 0 || lengthSec > 60 * 10) return "Length must be between 1 second and 10 minutes.";
    return null;
  };

  const handleGenerate = async () => {
    const err = validate();
    if (err) {
      addLog(`⚠️ Validation error: ${err}`);
      return;
    }

    setIsGenerating(true);
    setStatusLog([]);
    setResultUrl(null);
    addLog("Starting generation job...");

    try {
      const payload = {
        topic,
        ratio,
        lengthSec,
        providers: {
          sora: useSora,
          veo: useVeo,
          speechify: useSpeechify,
        },
        platformHints: ["tiktok", "youtube"], // used by backend to enforce guidelines
      };

      addLog("Sending request to /api/generate");

      const resp = await fetch("/api/generate", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload),
      });

      if (!resp.ok) {
        const text = await resp.text();
        addLog(`❌ Server error: ${resp.status} ${text}`);
        setIsGenerating(false);
        return;
      }

      // backend responds with { jobId } and the UI should poll /api/status/:jobId
      const { jobId } = await resp.json();
      addLog(`Job accepted: ${jobId}. Polling for status...`);

      // simple poll loop
      let attempts = 0;
      while (attempts < 240) {
        await new Promise((r) => setTimeout(r, 2000));
        attempts++;
        const st = await fetch(`/api/status/${encodeURIComponent(jobId)}`);
        if (!st.ok) {
          addLog(`Status error: ${st.status}`);
          break;
        }
        const j = await st.json();
        if (j.logs && j.logs.length) j.logs.forEach((l) => addLog(l));
        if (j.status === "done") {
          addLog("✅ Job finished");
          setResultUrl(j.resultUrl);
          setThumbnailPreview(j.thumbnailUrl || null);
          setCaption(j.caption || "");
          setIsGenerating(false);
          return;
        }
        if (j.status === "failed") {
          addLog(`❌ Job failed: ${j.error}`);
          setIsGenerating(false);
          return;
        }
        addLog(`Polling... (attempt ${attempts})`);
      }

      addLog("⚠️ Polling timed out. Check your server logs.");
      setIsGenerating(false);
    } catch (e) {
      console.error(e);
      addLog(`❌ Unexpected error: ${e.message}`);
      setIsGenerating(false);
    }
  };

  const downloadThumbnail = async () => {
    if (!thumbnailPreview) return;
    const a = document.createElement("a");
    a.href = thumbnailPreview;
    a.download = `thumbnail_${Date.now()}.png`;
    a.click();
  };

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-3xl font-semibold mb-4">One-Click AI Video Generator</h1>

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div className="p-4 border rounded-lg">
          <label className="block font-medium mb-2">Video topic (enter short topic or title)</label>
          <input
            value={topic}
            onChange={(e) => setTopic(e.target.value)}
            placeholder="e.g. What if humans never existed?"
            className="w-full border rounded p-2 mb-3"
          />

          <label className="block font-medium">Aspect ratio</label>
          <select value={ratio} onChange={(e) => setRatio(e.target.value)} className="w-full p-2 border rounded mb-3">
            <option value="16:9">16:9 (YouTube)</option>
            <option value="9:16">9:16 (TikTok / Shorts)</option>
            <option value="1:1">1:1 (Instagram)</option>
          </select>

          <label className="block font-medium">Length (seconds) — max 600 (10 minutes)</label>
          <input
            type="range"
            min={1}
            max={600}
            value={lengthSec}
            onChange={(e) => setLengthSec(Number(e.target.value))}
            className="w-full mb-2"
          />
          <div className="text-sm mb-3">Selected: {Math.floor(lengthSec / 60)}m {lengthSec % 60}s</div>

          <div className="flex items-center gap-3 mb-3">
            <label className="inline-flex items-center">
              <input type="checkbox" checked={useSora} onChange={(e) => setUseSora(e.target.checked)} className="mr-2" />
              Use Sora (visuals / storyboard)
            </label>
            <label className="inline-flex items-center">
              <input type="checkbox" checked={useVeo} onChange={(e) => setUseVeo(e.target.checked)} className="mr-2" />
              Use Veo (compositing / video builder)
            </label>
            <label className="inline-flex items-center">
              <input type="checkbox" checked={useSpeechify} onChange={(e) => setUseSpeechify(e.target.checked)} className="mr-2" />
              Use Speechify (narration)
            </label>
          </div>

          <div className="text-xs text-gray-600 mb-3">
            The backend must enforce community-guideline filters for TikTok/YouTube. This frontend only sends your request to the server.
          </div>

          <button
            onClick={handleGenerate}
            disabled={isGenerating}
            className={`px-4 py-2 rounded ${isGenerating ? "bg-gray-400" : "bg-blue-600 hover:bg-blue-700"} text-white`}
          >
            {isGenerating ? "Generating..." : "Generate One-Click Video"}
          </button>
        </div>

        <div className="p-4 border rounded-lg">
          <h2 className="font-semibold mb-2">Preview / Result</h2>
          <div className="mb-3">
            <strong>Thumbnail:</strong>
            <div className="mt-2">
              {thumbnailPreview ? (
                <div className="flex items-start gap-3">
                  <img src={thumbnailPreview} alt="thumbnail" className="w-48 rounded shadow" />
                  <div>
                    <button onClick={downloadThumbnail} className="px-3 py-1 border rounded mb-2 block">Download Thumbnail</button>
                    <div className="text-sm text-gray-600">Suggested size: 1280x720 (for 16:9) — backend will adapt to ratio.</div>
                  </div>
                </div>
              ) : (
                <div className="w-full h-40 bg-gray-100 rounded flex items-center justify-center text-gray-500">No thumbnail yet</div>
              )}
            </div>
          </div>

          <div className="mb-3">
            <strong>Caption (SEO & algorithm-friendly):</strong>
            <div className="mt-2 p-3 bg-gray-50 rounded min-h-[72px]">{caption || "No caption yet. Will be generated by backend."}</div>
          </div>

          <div className="mb-3">
            <strong>Final video URL:</strong>
            <div className="mt-2">{resultUrl ? <a href={resultUrl} target="_blank" rel="noreferrer" className="text-blue-600 underline">Open generated video</a> : "No result yet"}</div>
          </div>

          <div className="mb-3">
            <strong>Generation logs:</strong>
            <div className="mt-2 bg-black text-white p-2 rounded max-h-48 overflow-auto text-xs">
              {statusLog.length === 0 ? <div className="text-gray-400">Logs will appear here.</div> : statusLog.map((l, i) => <div key={i}>{l}</div>)}
            </div>
          </div>

        </div>
      </div>

      <div className="mt-6 p-4 border rounded-lg">
        <h3 className="font-semibold mb-2">Tips & notes</h3>
        <ul className="list-disc pl-5 text-sm text-gray-700">
          <li>Backend must perform moderation (text + images + audio) for TikTok/YouTube community guidelines before posting.</li>
          <li>Speechify offers TTS — use server-side integration to generate narration audio assets.</li>
          <li>Sora and Veo are referenced as visual/storyboard/video builder providers: connect to their APIs server-side and assemble clips into a final MP4/WebM.</li>
          <li>Thumbnail and caption are generated server-side: caption should include primary keyword, short hook, and hashtags tailored to the selected platform.</li>
        </ul>
      </div>
    </div>
  );
}

/* =====================
   Example backend (Node/Express) — put this on your server (do NOT put API keys in frontend)
   This is a condensed illustrative example. Replace the pseudo-calls with real vendor SDK/API calls.

// server.js (Node/Express)

const express = require('express');
const bodyParser = require('body-parser');
const fetch = require('node-fetch'); // or native fetch in newer Node
const crypto = require('crypto');

const app = express();
app.use(bodyParser.json({ limit: '5mb' }));

// In-memory job store (for demo only). In production use a DB + background worker.
const jobs = {};

function makeJobId() {
  return crypto.randomBytes(8).toString('hex');
}

// Basic moderation function — call an actual moderation API (OpenAI, Perspective, platform moderation)
async function moderateText(text) {
  // Example: call your moderation provider here.
  // Return { allowed: true } or { allowed: false, reason: 'hate' }
  return { allowed: true };
}

app.post('/api/generate', async (req, res) => {
  const { topic, ratio, lengthSec, providers, platformHints } = req.body;

  // Server-side validation
  if (!topic || typeof topic !== 'string') return res.status(400).send('Missing topic');
  if (lengthSec <= 0 || lengthSec > 600) return res.status(400).send('Invalid length');

  // Moderate topic
  const mod = await moderateText(topic);
  if (!mod.allowed) return res.status(403).json({ error: 'Topic blocked by moderation', reason: mod.reason });

  const jobId = makeJobId();
  jobs[jobId] = { status: 'queued', logs: ['Job queued'] };

  // Start background job (for demo: run immediately, but spawn a real background worker in prod)
  (async () => {
    try {
      jobs[jobId].status = 'running';
      jobs[jobId].logs.push('Starting asset generation');

      // 1) Generate script / storyboard from topic — call an LLM (GPT) on server-side
      // payload: topic + target length
      jobs[jobId].logs.push('Generating script (LLM)');
      // const script = await callYourLLM(`Write a ${Math.ceil(lengthSec/60)} minute script about ${topic}...`);
      const script = `Intro: What if...\nScene 1: ...\n(Generated script placeholder)`;

      // 2) Generate narration (Speechify) — TTS
      if (providers.speechify) {
        jobs[jobId].logs.push('Generating narration (Speechify)');
        // const ttsAudioUrl = await callSpeechifyAPI(script);
        const ttsAudioUrl = 'https://example.com/narration.mp3';
        jobs[jobId].narrationUrl = ttsAudioUrl;
      }

      // 3) Create visuals (Sora) — generate images or short clips for each scene
      if (providers.sora) {
        jobs[jobId].logs.push('Generating visuals (Sora)');
        // const visuals = await callSoraAPI(script, {ratio, lengthSec});
        const visuals = ['https://example.com/scene1.png', 'https://example.com/scene2.png'];
        jobs[jobId].visuals = visuals;
      }

      // 4) Compose video (Veo) — assemble visuals + audio + sfx into final mp4
      if (providers.veo) {
        jobs[jobId].logs.push('Compositing and rendering (Veo)');
        // const finalUrl = await callVeoCompose({visuals, narrationUrl, ratio, lengthSec});
        const finalUrl = 'https://example.com/final_video.mp4';
        jobs[jobId].resultUrl = finalUrl;
      }

      // 5) Generate thumbnail (use an image API or render a canvas)
      jobs[jobId].logs.push('Generating thumbnail');
      // const thumbUrl = await generateThumbnailFromVisuals(jobs[jobId].visuals);
      const thumbUrl = 'https://example.com/thumbnail.png';
      jobs[jobId].thumbnailUrl = thumbUrl;

      // 6) Make an algorithm-friendly caption (use LLM or templating) — include keyword, hook, CTA, hashtags
      jobs[jobId].logs.push('Generating caption (SEO)');
      const caption = `What if humans never existed? A 5-minute visual journey — Watch till the end. #whatif #science #shorts`;
      jobs[jobId].caption = caption;

      jobs[jobId].status = 'done';
      jobs[jobId].logs.push('Job complete');
    } catch (err) {
      console.error('Job error', err);
      jobs[jobId].status = 'failed';
      jobs[jobId].error = err.message;
      jobs[jobId].logs.push('Job failed: ' + err.message);
    }
  })();

  res.json({ jobId });
});

app.get('/api/status/:jobId', (req, res) => {
  const j = jobs[req.params.jobId];
  if (!j) return res.status(404).send('Not found');
  res.json({ status: j.status, logs: j.logs, resultUrl: j.resultUrl, thumbnailUrl: j.thumbnailUrl, caption: j.caption, error: j.error });
});

app.listen(3000, () => console.log('Server running on :3000'));


=== Integration notes ===
- Replace pseudo functions (callSpeechifyAPI, callSoraAPI, callVeoCompose) with actual provider SDK/API calls.
- Respect rate limits and file size limits. Transcoding/rendering can be resource heavy — use asynchronous background workers (queues like Bull / Redis, or cloud functions + storage).
- Moderation: always moderate text and images before composing final video. For TikTok/YouTube, also ensure music licensing and avoid disallowed content.
- For captions: produce 2 variants — short (<=100 chars) for TikTok and longer for YouTube. Include relevant hashtags and your primary keyword in the first 3 words.

*/
