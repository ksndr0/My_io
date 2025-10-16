# My_io
One-click AI video generator that creates full videos with narration, visuals, and captions.
import React, { useState } from "react";

// One-Click AI Video Generator
// - Single-file React component (TailwindCSS classes used for styling)
// - Frontend UI for: topic input, ratio selection, length (max 10 minutes), provider toggles (Sora, Veo, Speechify)
// - One-click calls a backend endpoint (/api/generate) that orchestrates AI services
// - The backend snippet (Node/Express) is included at the bottom as a commented block. You MUST implement server-side logic
//   for API keys, moderation, and to call providers' APIs securely.
//
// How to use:
// 1) Install Tailwind + React in your project (or paste this into a create-react-app + Tailwind setup).
// 2) Create a server endpoint /api/generate (example Node snippet below) and add your provider API keys there.
// 3) Deploy frontend and backend. Frontend calls backend which returns a { jobId, status, resultUrl } or streams progress.
//
// Important: This component does not contain secret keys and does no real AI calls — backend must do them.

export default function OneClickVideoGenerator() {
  const [topic, setTopic] = useState("");
  const [ratio, setRatio] = useState("16:9");
  const [lengthSec, setLengthSec] = useState(60 * 2); // default 2 minutes
  const [useSora, setUseSora] = useState(true);
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
