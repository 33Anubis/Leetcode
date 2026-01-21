// ==UserScript==
// @name         Freshchat & Freshdesk AI Scribe
// @namespace    local
// @version      0.7
// @match        https://support.empathia.ai/*
// @match        https://empathiaai.freshdesk.com/a/tickets/*
// @match        https://empathia-ai.myfreshworks.com/crm/messaging/*
// @grant        GM_xmlhttpRequest
// @connect      empathiaai.freshdesk.com
// @connect      empathia-ai.myfreshworks.com
// @connect      generativelanguage.googleapis.com
// ==/UserScript==

(() => {
  'use strict';

  const STORAGE_KEY = 'fd_ai_cfg_v1';

  const ui = mountUI();

  // Auto-load config on startup
  const savedConfig = localStorage.getItem(STORAGE_KEY);
  if (savedConfig) {
      try {
          ui.setConfig(JSON.parse(savedConfig));
          ui.setStatus('Config auto-loaded ✅');
      } catch (e) {
          ui.setStatus('Auto-load failed ❌');
      }
  }

  updateTicketMeta();
  hookUrlChanges(updateTicketMeta);

  ui.onSave(() => {
    const cfg = ui.getConfig();
    cfg.freshdeskDomain = normalizeHost(cfg.freshdeskDomain);
    localStorage.setItem(STORAGE_KEY, JSON.stringify(cfg));
    ui.setStatus('Saved ✅');
  });

    ui.onInsert(() => {
  const text = ui.getReply();
  if (!text) return ui.setStatus('No reply to insert ❌');

  try {
    injectTextIntoFreshdesk(text);
    ui.setStatus('Inserted ✅');
  } catch (e) {
    ui.setStatus(e.message);
  }
});

  ui.onLoad(() => {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return ui.setStatus('Nothing saved yet.');
    try {
      ui.setConfig(JSON.parse(raw));
      ui.setStatus('Loaded ✅');
    } catch {
      ui.setStatus('Failed to load config ❌');
    }
  });

  function normalizeHost(input) {
    return String(input || '')
      .trim()
      .replace(/^https?:\/\//i, '')
      .replace(/\/.*$/, '');
  }

async function getFreshchatAlias(domain, apiKey, numericId) {
    // This is the internal fetch endpoint we identified in your network logs
    const url = `https://${domain}/crm/messaging/app/agent/inbox/conversation/fetch?appId=780320984368207&ids=${numericId}`;
    
    const res = await gmGetJson(url, {
        'Authorization': `Bearer ${apiKey}`,
        'Accept': 'application/json'
    });

    if (res.conversations && res.conversations.length > 0) {
        return res.conversations[0].alias; // This is the UUID alias
    }
    throw new Error("Could not map Numeric ID to API Alias.");
}
    
  ui.onGenerate(async () => {
    ui.setStatus('Fetching ticket…');
    ui.setReply('');

    const ticketId = getTicketIdFromUrl();
    const isFreshchat = window.location.href.includes('crm/messaging');
    if (!ticketId) return ui.setStatus('Open a ticket page first.');

    const cfg = ui.getConfig();
    if (!cfg.freshdeskDomain || !cfg.freshdeskApiKey) {
      return ui.setStatus('Missing Freshdesk domain or API key.');
    }

    try {
      let thread = "";
        let subject = "Chat Conversation";

        if (isFreshchat) {
            const numericId = window.location.href.split('/').pop();
            // 1. Get the Alias
            const alias = await getFreshchatAlias(cfg.freshdeskDomain, cfg.freshdeskApiKey, numericId);
            
            // 2. Fetch Freshchat Messages using Alias
            const chatUrl = `https://${cfg.freshdeskDomain}/v2/conversations/${alias}/messages`;
            const chatData = await gmGetJson(chatUrl, { 'Authorization': `Bearer ${cfg.freshdeskApiKey}` });
            
            // 3. Parse Freshchat message structure
            thread = chatData.messages
                .slice(-10)
                .map(m => {
                    const actor = m.actor_type;
                    const content = m.message_parts?.[0]?.text?.content || "";
                    return `${actor}: ${content}`;
                })
                .join('\n');
        } else {
            const ticket = await fetchTicket(cfg.freshdeskDomain, cfg.freshdeskApiKey, ticketId);
      const convos = await fetchConversations(cfg.freshdeskDomain, cfg.freshdeskApiKey, ticketId);

      const subject = ticket?.subject ?? '(no subject)';
      const desc = ticket?.description_text ?? stripHtml(ticket?.description ?? '') ?? '(no description)';

      function logTicket(ticketId){
          console.log(fetchConversations(cfg.freshdeskDomain, cfg.freshdeskApiKey, ticketId));
      }
        
      const thread = (Array.isArray(convos) ? convos : [])
        .filter(c => !c.private)
        .slice(-5)
        .map(c => {
          const from = c?.from_email || c?.from_name || 'unknown';
          const body = c?.body_text || stripHtml(c?.body || '');
          return `From: ${from}\n${body}`.slice(0, 2000);
        })
        .join('\n\n---\n\n');

      if (!cfg.geminiApiKey) return ui.setStatus('Missing Gemini API key.');

      ui.setStatus('Drafting reply…');
        }
      

      const prompt = `
You are a helpful customer support agent replying inside Freshdesk.

Rules:
- Do NOT mention you are an AI.
- Be concise, friendly, and actionable.
- If info is missing, ask up to 2 crisp questions.
- If you propose steps, give them as a short checklist.

Ticket subject: ${subject}

Ticket description:
${desc}

Recent thread:
${thread || '(no recent messages)'}

Extra notes from me:
${ui.getNotes() || '(none)'}

Write the final reply ONLY (no preamble, no analysis).
`.trim();

      const reply = await callGemini(cfg.geminiApiKey, cfg.geminiModel || 'gemini-2.5-flash', prompt);

      ui.setReply(reply);
      ui.setStatus('Done ✅');

    } catch (e) {
      ui.setStatus(`Fetch failed ❌ ${String(e.message || e)}`);
    }
  });

  ui.onCopy(async () => {
    const text = ui.getReply().trim();
    if (!text) return ui.setStatus('Nothing to copy.');
    await navigator.clipboard.writeText(text);
    ui.setStatus('Copied ✅');
  });

  function updateTicketMeta() {
    const id = getTicketIdFromUrl();
    ui.setMeta(id ? `Ticket: ${id}` : 'Not on a ticket page');
  }

  function getTicketIdFromUrl() {
    const m = location.pathname.match(/\/(?:a\/)?tickets\/(\d+)/i);
    return m ? m[1] : null;
  }
  

  function hookUrlChanges(onChange) {
    const _pushState = history.pushState;
    const _replaceState = history.replaceState;

    history.pushState = function () {
      _pushState.apply(this, arguments);
      onChange();
    };
    history.replaceState = function () {
      _replaceState.apply(this, arguments);
      onChange();
    };
    window.addEventListener('popstate', onChange);
  }

  // ===== Freshdesk API =====

  function basicAuth(apiKey) {
    return `Basic ${btoa(`${apiKey}:X`)}`;
  }

  function gmGetJson(url, headers) {
    return new Promise((resolve, reject) => {
      GM_xmlhttpRequest({
        method: 'GET',
        url,
        headers,
        onload: (res) => {
          if (res.status >= 200 && res.status < 300) {
            try {
              resolve(JSON.parse(res.responseText));
            } catch {
              reject(new Error('Failed to parse JSON response.'));
            }
          } else {
            reject(new Error(`${res.status} ${res.statusText}\n${res.responseText}`));
          }
        },
        onerror: () => reject(new Error('Network error (GM_xmlhttpRequest).')),
      });
    });
  }

  function fetchTicket(domain, apiKey, ticketId) {
    const url = `https://${domain}/api/v2/tickets/${ticketId}`;
    return gmGetJson(url, {
      Authorization: basicAuth(apiKey),
      'Content-Type': 'application/json',
    });
  }

  function fetchConversations(domain, apiKey, ticketId) {
    const url = `https://${domain}/api/v2/tickets/${ticketId}/conversations`;
    return gmGetJson(url, {
      Authorization: basicAuth(apiKey),
      'Content-Type': 'application/json',
    });
  }

  function gmPostJson(url, headers, bodyObj) {
    return new Promise((resolve, reject) => {
      GM_xmlhttpRequest({
        method: 'POST',
        url,
        headers: { 'Content-Type': 'application/json', ...headers },
        data: JSON.stringify(bodyObj),
        onload: (res) => {
          if (res.status >= 200 && res.status < 300) {
            try {
              resolve(JSON.parse(res.responseText));
            } catch {
              reject(new Error('Failed to parse Gemini JSON.'));
            }
          } else {
            reject(new Error(`${res.status} ${res.statusText}\n${res.responseText}`));
          }
        },
        onerror: () => reject(new Error('Network error calling Gemini.')),
      });
    });
  }

  async function callGemini(apiKey, model, promptText) {
    const url = `https://generativelanguage.googleapis.com/v1beta/models/${encodeURIComponent(model)}:generateContent`;
    const data = await gmPostJson(url, { 'x-goog-api-key': apiKey }, {
      contents: [{ role: 'user', parts: [{ text: promptText }] }],
    });

    const text = (data?.candidates?.[0]?.content?.parts || [])
      .map(p => p?.text)
      .filter(Boolean)
      .join('\n')
      .trim();

    if (!text) throw new Error('Gemini returned no text.');
    return text;
  }

  function stripHtml(html) {
    return String(html || '')
      .replace(/<style[\s\S]*?>[\s\S]*?<\/style>/gi, '')
      .replace(/<script[\s\S]*?>[\s\S]*?<\/script>/gi, '')
      .replace(/<\/p>/gi, '\n')
      .replace(/<br\s*\/?>/gi, '\n')
      .replace(/<[^>]+>/g, '')
      .replace(/&nbsp;/g, ' ')
      .replace(/&amp;/g, '&')
      .replace(/&lt;/g, '<')
      .replace(/&gt;/g, '>')
      .replace(/\n{3,}/g, '\n\n')
      .trim();
  }
    function injectTextIntoFreshdesk(text) {
  // 1. Try to find the editor (Freshdesk usually uses a div with 'redactor-editor' or 'contenteditable')
  let editor = document.querySelector('.redactor-editor') ||
               document.querySelector('[contenteditable="true"]');

  if (!editor) {
    // 2. If not found, try to click the "Reply" button automatically
    const replyBtn = document.querySelector('[data-test-id="ticket-action-reply"]');
    if (replyBtn) {
      replyBtn.click();
      // Wait a moment for the editor to mount, then try again
      setTimeout(() => injectTextIntoFreshdesk(text), 500);
      return;
    }
    throw new Error("Could not find reply box. Please click 'Reply' first.");
  }

  // 3. Inject the text.
  // Freshdesk often uses <p> tags. We'll convert newlines to <p> tags for better formatting.
  const htmlContent = text.split('\n').map(line => `<p>${line}</p>`).join('');

  editor.focus();
  editor.innerHTML = htmlContent;

  // 4. Trigger an 'input' event so Freshdesk knows the content changed
  editor.dispatchEvent(new Event('input', { bubbles: true }));
}

  // ===== UI =====

function mountUI() {
    const UI_STATE_KEY = 'fd_ai_ui_state_v1';

    const wrap = document.createElement('div');
    wrap.id = 'ai-scribe-panel';
    wrap.style.cssText = `
      position: fixed; right: 16px; bottom: 16px; width: 420px; max-height: 75vh;
      overflow: auto; background: #111; color: #eee; border: 1px solid #333;
      border-radius: 12px; padding: 12px; z-index: 999999;
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial;
      box-shadow: 0 10px 35px rgba(0,0,0,.4);
    `;

    wrap.innerHTML = `
      <div id="panel">
        <div style="display:flex;justify-content:space-between;align-items:center;gap:8px;">
          <div>
            <div style="font-weight:700;">AI Scribe</div>
            <div id="meta" style="font-size:12px;color:#aaa;">—</div>
          </div>
          <div style="display:flex;gap:6px;align-items:center;">
            <button id="load">load</button>
            <button id="save">save</button>
            <button id="collapse" title="Collapse" aria-label="Collapse">✕</button>
          </div>
        </div>

        <div id="status" style="margin-top:8px;font-size:12px;color:#aaa;">Ready</div>

        <div style="display:grid;gap:6px;margin-top:10px;">
          <input id="fdDomain" placeholder="Freshdesk domain (e.g. support.yourcompany.ai)" />
          <input id="fdKey" placeholder="Freshdesk API key" />
          <input id="gmKey" placeholder="Gemini API key" />
          <input id="gmModel" placeholder="Gemini model" value="gemini-2.5-flash"/>
        </div>

        <textarea id="notes" placeholder="Extra notes for this specific reply..."
          style="width:100%;margin-top:10px;min-height:80px;background:#0b0b0b;color:#eee;border:1px solid #2a2a2a;border-radius:10px;padding:8px 10px;"></textarea>

        <div style="display:flex;gap:8px;margin-top:10px;">
          <button id="generate" style="font-weight:700;">Generate Draft</button>
          <button id="insert">Insert</button> <button id="dictate" title="Dictate notes">🎤 Dictate</button>
          <button id="copy">Copy Reply</button>
        </div>

        <textarea id="reply" placeholder="AI draft will appear here..."
          style="width:100%;margin-top:10px;min-height:140px;background:#0b0b0b;color:#eee;border:1px solid #2a2a2a;border-radius:10px;padding:8px 10px;" readonly></textarea>
      </div>

      <button id="bubble" title="Open AI Scribe" aria-label="Open AI Scribe"
        style="
          display:none; position: fixed; right: 16px; bottom: 16px; width: 54px; height: 54px;
          border-radius: 999px; background: #111; color: #eee; border: 1px solid #333;
          box-shadow: 0 10px 35px rgba(0,0,0,.4); cursor: pointer; font-size: 18px;
        ">🤖</button>
    `;

    const panel = wrap.querySelector('#panel');
    [...panel.querySelectorAll('button')].forEach((b) => {
      b.style.cssText = 'background:#1a1a1a;color:#eee;border:1px solid #2a2a2a;border-radius:10px;padding:7px 10px;cursor:pointer;font-size:13px;';
    });
    [...panel.querySelectorAll('input')].forEach((i) => {
      i.style.cssText = 'width:100%;background:#0b0b0b;color:#eee;border:1px solid #2a2a2a;border-radius:10px;padding:8px 10px;font-size:13px;';
    });

    document.body.appendChild(wrap);

    const $ = (sel) => wrap.querySelector(sel);
    const meta = $('#meta');
    const status = $('#status');
    const fdDomain = $('#fdDomain');
    const fdKey = $('#fdKey');
    const gmKey = $('#gmKey');
    const gmModel = $('#gmModel');
    const notes = $('#notes');
    const reply = $('#reply');
    const bubble = $('#bubble');
    const dictateBtn = $('#dictate');

    function setCollapsed(collapsed) {
      panel.style.display = collapsed ? 'none' : 'block';
      bubble.style.display = collapsed ? 'block' : 'none';
      wrap.style.width = collapsed ? '0' : '420px';
      wrap.style.height = collapsed ? '0' : 'auto';
      wrap.style.padding = collapsed ? '0' : '12px';
      wrap.style.background = collapsed ? 'transparent' : '#111';
      wrap.style.boxShadow = collapsed ? 'none' : '0 10px 35px rgba(0,0,0,.4)';
      localStorage.setItem(UI_STATE_KEY, JSON.stringify({ collapsed }));
    }

    $('#collapse').addEventListener('click', () => setCollapsed(true));
    bubble.addEventListener('click', () => setCollapsed(false));

    try {
      const saved = JSON.parse(localStorage.getItem(UI_STATE_KEY) || '{}');
      if (saved.collapsed) setCollapsed(true);
    } catch {}

    // --- Dictation Logic (Moved BEFORE return) ---
    let recognition = null;
    function stopDictation() {
      dictateBtn.textContent = '🎤 Dictate';
      dictateBtn.style.background = '#1a1a1a';
    }

    if ('webkitSpeechRecognition' in window || 'SpeechRecognition' in window) {
      const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
      recognition = new SpeechRecognition();
      recognition.continuous = false;
      recognition.interimResults = false;
      recognition.lang = 'en-US';

      recognition.onstart = () => {
        dictateBtn.textContent = '🛑 Listening...';
        dictateBtn.style.background = '#4a1111';
      };

      recognition.onresult = (event) => {
        const transcript = event.results[0][0].transcript;
        notes.value = notes.value ? `${notes.value} ${transcript}` : transcript;
        status.textContent = 'Voice captured ✅';
      };

      recognition.onerror = (e) => {
        status.textContent = `Speech error: ${e.error}`;
        stopDictation();
      };

      recognition.onend = stopDictation;

      dictateBtn.addEventListener('click', () => {
        if (dictateBtn.textContent.includes('Listening')) {
          recognition.stop();
        } else {
          recognition.start();
        }
      });
    } else {
      dictateBtn.disabled = true;
      dictateBtn.title = "Speech recognition not supported.";
    }

    // --- The Return Statement ---
    return {
      setMeta: (t) => (meta.textContent = t),
      setStatus: (t) => (status.textContent = t),
      getConfig: () => ({
        freshdeskDomain: fdDomain.value.trim(),
        freshdeskApiKey: fdKey.value.trim(),
        geminiApiKey: gmKey.value.trim(),
        geminiModel: gmModel.value.trim() || 'gemini-2.0-flash',
      }),
      setConfig: (cfg) => {
        fdDomain.value = cfg.freshdeskDomain || '';
        fdKey.value = cfg.freshdeskApiKey || '';
        gmKey.value = cfg.geminiApiKey || '';
        gmModel.value = cfg.geminiModel || 'gemini-2.0-flash';
      },
      getNotes: () => notes.value || '',
      setReply: (t) => (reply.value = t || ''),
      getReply: () => reply.value || '',
      onSave: (fn) => $('#save').addEventListener('click', fn),
      onLoad: (fn) => $('#load').addEventListener('click', fn),
      onGenerate: (fn) => $('#generate').addEventListener('click', fn),
      onCopy: (fn) => $('#copy').addEventListener('click', fn),
      onInsert: (fn) => $('#insert').addEventListener('click', fn),
    };
  }
})();