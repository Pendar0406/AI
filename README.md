// AI-chat-webpage
// En enkel, färdig React-komponent (single-file) som visar en liten chat-app
// Frontend anropar ett exempel-API (/api/chat) som du kör lokalt (Node/Express) som proxy till OpenAI.
// Viktigt: lägg aldrig din OpenAI-nyckel i frontend. Kör nyckeln i backend (server).

// --------------------------
// Installation (kort):
// 1) Skapa ett nytt Vite React-projekt: `npm create vite@latest my-ai-app -- --template react`
// 2) Gå in i mappen: `cd my-ai-app` och installera beroenden: `npm install`
// 3) Lägg till Tailwind (valfritt) eller bara kör med vanlig CSS.
// 4) Kopiera denna komponent till src/App.jsx och skapa backend-filen server/index.js enligt nedan.
// 5) Kör backend: `node server/index.js` och kör frontend: `npm run dev`.

/*
==================== FRONTEND: src/App.jsx ====================
*/
import React, { useState, useRef } from "react";

export default function App() {
  const [messages, setMessages] = useState([
    { role: "system", text: "Du är en hjälpsam AI-assistent." },
    { role: "assistant", text: "Hej! Säg hej och ställ en fråga — jag svarar." },
  ]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const bottomRef = useRef(null);

  const send = async () => {
    if (!input.trim()) return;
    const userMsg = { role: "user", text: input };
    setMessages((m) => [...m, userMsg]);
    setInput("");
    setLoading(true);

    try {
      // Skickar konversationen till backend som i sin tur anropar OpenAI
      const resp = await fetch("/api/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ messages: [...messages, userMsg] }),
      });

      if (!resp.ok) throw new Error(`Serverfel: ${resp.status}`);
      const data = await resp.json();
      // Förväntar oss { reply: "text..." }
      setMessages((m) => [...m, { role: "assistant", text: data.reply }]);
    } catch (e) {
      console.error(e);
      setMessages((m) => [...m, { role: "assistant", text: "Förlåt, något gick fel. Prova igen senare." }]);
    } finally {
      setLoading(false);
      bottomRef.current?.scrollIntoView({ behavior: "smooth" });
    }
  };

  const handleKey = (e) => {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      send();
    }
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-sky-50 p-6 flex items-center justify-center">
      <div className="w-full max-w-2xl bg-white/90 backdrop-blur-md shadow-lg rounded-2xl p-6">
        <header className="flex items-center justify-between mb-4">
          <h1 className="text-2xl font-semibold">Min AI-chat</h1>
          <div className="text-sm text-slate-500">Frontend + enkel backend</div>
        </header>

        <main className="space-y-4 max-h-[60vh] overflow-y-auto p-2 rounded" aria-live="polite">
          {messages.map((m, i) => (
            <div key={i} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
              <div className={`rounded-xl p-3 max-w-[80%] ${m.role === 'user' ? 'bg-sky-500 text-white' : 'bg-slate-100 text-slate-900'}`}>
                <div className="text-sm whitespace-pre-wrap">{m.text}</div>
                <div className="text-xs opacity-60 mt-1">{m.role}</div>
              </div>
            </div>
          ))}
          <div ref={bottomRef} />
        </main>

        <footer className="mt-4">
          <div className="flex gap-2">
            <textarea
              value={input}
              onChange={(e) => setInput(e.target.value)}
              onKeyDown={handleKey}
              placeholder="Skriv din fråga här... (Enter för att skicka)"
              className="flex-1 p-3 rounded-lg border border-slate-200 resize-none h-16"
            />
            <button
              onClick={send}
              disabled={loading}
              className="px-4 py-2 rounded-lg bg-sky-600 text-white font-medium disabled:opacity-60"
            >
              {loading ? "Skickar..." : "Skicka"}
            </button>
          </div>
          <div className="text-xs text-slate-500 mt-2">Tips: Bygg backend så håller du din API-nyckel hemlig.</div>
        </footer>
      </div>
    </div>
  );
}

/*
==================== BACKEND: server/index.js (Node/Express) ====================
// Kör `npm init -y` och `npm i express node-fetch dotenv` i server-mappen.
// Skapa en .env-fil i server/ med OPENAI_API_KEY=din_nyckel (lägg aldrig nyckeln i frontend)

const express = require('express');
const fetch = require('node-fetch');
const dotenv = require('dotenv');
const path = require('path');

dotenv.config();
const app = express();
app.use(express.json());

// För produktion: servera frontend-build (valfritt)
// app.use(express.static(path.join(__dirname, '../dist')));

app.post('/api/chat', async (req, res) => {
  try {
    const { messages } = req.body;
    // Bygg prompt för OpenAI (exempel för Chat Completions)

    // OBS: Exakt endpoint och payload beror på vilken OpenAI-modell/SDK du använder.
    // Här använder vi ett enkelt fetch-anrop till gpt-4o-mini eller annan kompatibel endpoint.

    const payload = {
      model: 'gpt-4o-mini',
      messages: messages.map(m => ({ role: m.role, content: m.text })),
      max_tokens: 300,
    };

    const r = await fetch('https://api.openai.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify(payload),
    });

    if (!r.ok) {
      const text = await r.text();
      console.error('OpenAI error', r.status, text);
      return res.status(502).json({ error: 'Fel från OpenAI' });
    }

    const data = await r.json();
    // Anpassa beroende på svar
    const reply = data.choices?.[0]?.message?.content ?? 'Inget svar från model.';
    res.json({ reply });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'Serverfel' });
  }
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => console.log(`Servern körs på port ${PORT}`));

*/

/*
==================== Viktiga säkerhets-noteringar ====================
- Lagra aldrig API-nycklar i frontend eller på GitHub offentligt.
- Begränsa användningen av din server (rate limiting, auth) om du öppnar för allmänheten.
- Du kan också använda serverless-funktioner (Vercel, Netlify) som proxy.

==================== Vill du vidare? ====================
Om du vill kan jag:
- Anpassa layouten (färger, profilbild, tema)
- Göra ett komplett GitHub-repo / package.json och Tailwind-setup
- Bygga en enklare mock-AI (utan OpenAI) för offline-testning

Säg vad du vill ändra så uppdaterar jag koden.
*/
