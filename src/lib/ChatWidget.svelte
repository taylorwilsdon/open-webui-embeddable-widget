<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { marked } from 'marked';

  // --- state --------------------------------------------------------------
  let messages = $state<{ role: string; content: string }[]>([]);
  let input = $state('');
  let isLoading = $state(false);
  let messagesContainer: HTMLDivElement;

  // Configuration variables, set once onMount. Not $state.
  let apiKey: string = '';
  let model = $state('gpt-4o-mini');     // default fallback
  let modelName = $state('gpt-4o-mini');     // default fallback
  let endpoint: string = '/api/chat/completions'; // relative to current host by default

  // Configure marked for secure rendering
  marked.setOptions({
    breaks: true,
    gfm: true
  });

  // Function to safely render markdown
  function renderMarkdown(content: string): string {
    return marked.parse(content) as string;
  }

  // --- initialize from query‑string --------------------------------------
  onMount(() => {
    console.log('ChatWidget onMount triggered. Attempting to read query parameters.');
    try {
      const qs = new URLSearchParams(window.location.search);
      const qsApiKey = qs.get('api_key');
      const qsModel = qs.get('model');
      const qsModelName = qs.get('model_name');
      const qsEndpoint = qs.get('endpoint');

      if (qsApiKey !== null) apiKey = qsApiKey;
      if (qsModel !== null) model = qsModel;
      if (qsModelName !== null) modelName = qsModelName;
      else modelName = model;
      if (qsEndpoint !== null) endpoint = qsEndpoint;
      // Default values are already set if query params are null

      console.log('Query parameters processed:', { apiKey, model, endpoint });
    } catch (e) {
      console.error('Error processing query parameters in onMount:', e);
    }
  });

  // Auto-scroll to bottom when new messages arrive
  $effect(() => {
    if (messages.length && messagesContainer) {
      setTimeout(() => {
        messagesContainer.scrollTop = messagesContainer.scrollHeight;
      }, 100);
    }
  });

  // --- helpers ------------------------------------------------------------
  async function send() {
    const text = input.trim();
    if (!text || isLoading) return;
    const currentFocus = document.activeElement;

    // optimistic UI
    messages = [...messages, { role: 'user', content: text }];
    input = '';
    isLoading = true;

    try {
      const res = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${apiKey}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({
          model,
          messages //: [{ role: 'user', content: text }]
        })
      });
      const data = await res.json();
      const reply = data?.choices?.[0]?.message?.content ?? '⚠️ Error retrieving response';
      messages = [...messages, { role: 'assistant', content: reply }];
    } catch (err) {
      messages = [...messages, { role: 'assistant', content: '⚠️ Network error' }];
    } finally {
      isLoading = false;
      setTimeout(() => currentFocus.focus(), 100);
    }
  }
  function clear() {
    messages = [];
  }

  function enterToSend(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      send();
    }
  }

  function adjustTextareaHeight(e: Event) {
    const textarea = e.target as HTMLTextAreaElement;
    textarea.style.height = 'auto';
    textarea.style.height = Math.min(textarea.scrollHeight, 120) + 'px';
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600&display=swap');

  :global(*) {
    box-sizing: border-box;
  }

  .wrapper {
    display: flex;
    flex-direction: column;
    height: 100%;
    background: #ffffff;
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Helvetica Neue', Arial, sans-serif;
    color: #202123;
    border-radius: 8px;
    overflow: hidden;
    box-shadow: 0 0 0 1px rgba(0, 0, 0, 0.05), 0 2px 4px rgba(0, 0, 0, 0.05);
  }

  .header {
    padding: 1rem 1.25rem;
    background: #ffffff;
    border-bottom: 1px solid #e5e5e5;
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .header-icon {
    width: 28px;
    height: 28px;
    background: #10a37f;
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-weight: 600;
    font-size: 16px;
  }

  .header-title {
    font-size: 1rem;
    font-weight: 600;
    color: #202123;
  }
  .header-clear {
    width: 100%;
    text-align: right;
  }

  .messages {
    flex: 1;
    overflow-y: auto;
    padding: 1.5rem 0;
    background: #ffffff;
    scroll-behavior: smooth;
  }

  .messages::-webkit-scrollbar {
    width: 8px;
  }

  .messages::-webkit-scrollbar-track {
    background: transparent;
  }

  .messages::-webkit-scrollbar-thumb {
    background: #c5c5d2;
    border-radius: 4px;
  }

  .messages::-webkit-scrollbar-thumb:hover {
    background: #9a9aa8;
  }

  .message-wrapper {
    display: flex;
    gap: 1rem;
    padding: 1rem 1.5rem;
    animation: fadeIn 0.3s ease-out;
  }

  .message-wrapper.user {
    background: #f7f7f8;
  }

  .message-wrapper.assistant {
    background: #ffffff;
  }

  .avatar {
    width: 30px;
    height: 30px;
    border-radius: 2px;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 600;
    font-size: 13px;
    color: white;
  }

  .avatar.user {
    background: #5436da;
  }

  .avatar.assistant {
    background: #10a37f;
  }

  .message {
    flex: 1;
    line-height: 1.75;
    font-size: 0.875rem;
    color: #202123;
    word-wrap: break-word;
    animation: slideIn 0.3s ease-out;
  }

  /* Markdown content styling - :global() to avoid unused selector warnings */
  :global(.markdown-content h1),
  :global(.markdown-content h2),
  :global(.markdown-content h3),
  :global(.markdown-content h4),
  :global(.markdown-content h5),
  :global(.markdown-content h6) {
    font-weight: 600;
    margin: 1em 0 0.5em 0;
    line-height: 1.4;
  }

  :global(.markdown-content h1) { font-size: 1.25rem; }
  :global(.markdown-content h2) { font-size: 1.125rem; }
  :global(.markdown-content h3) { font-size: 1rem; }
  :global(.markdown-content h4) { font-size: 0.9rem; }
  :global(.markdown-content h5) { font-size: 0.875rem; }
  :global(.markdown-content h6) { font-size: 0.875rem; font-weight: 500; }

  :global(.markdown-content p) {
    margin: 0.5em 0;
  }

  :global(.markdown-content strong) {
    font-weight: 600;
  }

  :global(.markdown-content em) {
    font-style: italic;
  }

  :global(.markdown-content code) {
    background: #f6f8fa;
    border: 1px solid #e1e4e8;
    border-radius: 3px;
    padding: 0.125em 0.25em;
    font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
    font-size: 0.8125rem;
    color: #24292e;
  }

  :global(.markdown-content pre) {
    background: #f6f8fa;
    border: 1px solid #e1e4e8;
    border-radius: 6px;
    padding: 1rem;
    margin: 0.75em 0;
    overflow-x: auto;
    line-height: 1.5;
  }

  :global(.markdown-content pre code) {
    background: none;
    border: none;
    padding: 0;
    font-size: 0.8125rem;
  }

  :global(.markdown-content blockquote) {
    border-left: 4px solid #e1e4e8;
    padding-left: 1rem;
    margin: 0.75em 0;
    color: #656d76;
    font-style: italic;
  }

  :global(.markdown-content ul),
  :global(.markdown-content ol) {
    margin: 0.5em 0;
    padding-left: 1.5rem;
  }

  :global(.markdown-content li) {
    margin: 0.25em 0;
  }

  :global(.markdown-content a) {
    color: #10a37f;
    text-decoration: underline;
  }

  :global(.markdown-content a:hover) {
    color: #0d8f6b;
  }

  :global(.markdown-content hr) {
    border: none;
    border-top: 1px solid #e1e4e8;
    margin: 1.5em 0;
  }

  :global(.markdown-content table) {
    border-collapse: collapse;
    margin: 0.75em 0;
    width: 100%;
  }

  :global(.markdown-content th),
  :global(.markdown-content td) {
    border: 1px solid #e1e4e8;
    padding: 0.375rem 0.75rem;
    text-align: left;
  }

  :global(.markdown-content th) {
    background: #f6f8fa;
    font-weight: 600;
  }

  .loading-dots {
    display: flex;
    gap: 3px;
    padding: 0.5rem 0;
  }

  .loading-dot {
    width: 6px;
    height: 6px;
    background: #8e8ea0;
    border-radius: 50%;
    animation: bounce 1.4s infinite ease-in-out both;
  }

  .loading-dot:nth-child(1) {
    animation-delay: -0.32s;
  }

  .loading-dot:nth-child(2) {
    animation-delay: -0.16s;
  }

  .input-container {
    padding: 1rem 1.5rem 1.25rem;
    background: #ffffff;
    border-top: 1px solid #e5e5e5;
  }

  .input-wrapper {
    display: flex;
    align-items: flex-end;
    gap: 0.75rem;
    background: #ffffff;
    border: 1px solid #d9d9e3;
    border-radius: 6px;
    padding: 0.75rem 0.75rem 0.75rem 1rem;
    transition: border-color 0.2s ease;
  }

  .input-wrapper:focus-within {
    border-color: #10a37f;
  }

  textarea {
    flex: 1;
    border: none;
    outline: none;
    resize: none;
    font-family: inherit;
    font-size: 0.875rem;
    line-height: 1.5;
    color: #202123;
    background: transparent;
    min-height: 20px;
    max-height: 120px;
    padding: 0;
  }

  textarea::placeholder {
    color: #8e8ea0;
  }

  button {
    padding: 0.375rem;
    border: none;
    border-radius: 4px;
    background: #ffffff;
    color: #10a37f;
    cursor: pointer;
    transition: all 0.15s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    flex-shrink: 0;
  }

  button:hover:not(:disabled) {
    color: #202123;
  }

  button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
    color: #8e8ea0;
  }

  .send-icon {
    width: 16px;
    height: 16px;
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes slideIn {
    from {
      transform: translateY(5px);
      opacity: 0;
    }
    to {
      transform: translateY(0);
      opacity: 1;
    }
  }

  @keyframes bounce {
    0%, 80%, 100% {
      transform: scale(0);
    }
    40% {
      transform: scale(1);
    }
  }
</style>

<div class="wrapper">
  <div class="header">
    <div class="header-icon">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M12 2L2 7L12 12L22 7L12 2Z"></path>
        <path d="M2 17L12 22L22 17"></path>
        <path d="M2 12L12 17L22 12"></path>
      </svg>
    </div>
    <div class="header-title">{modelName}</div>
    <div class="header-clear"><button onclick={clear} style="cursor: pointer;float: right;">🗑️</button></div>
  </div>
  <div class="messages" bind:this={messagesContainer}>
    {#each messages as message}
      <div class="message-wrapper {message.role}">
        <div class="avatar {message.role}">
          {#if message.role === 'user'}
            <svg width="16" height="16" viewBox="0 0 24 24" fill="currentColor">
              <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
              <circle cx="12" cy="7" r="4"></circle>
            </svg>
          {:else}
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M12 2L2 7L12 12L22 7L12 2Z"></path>
              <path d="M2 17L12 22L22 17"></path>
              <path d="M2 12L12 17L22 12"></path>
            </svg>
          {/if}
        </div>
        <div class="message markdown-content">
          {@html renderMarkdown(message.content)}
        </div>
      </div>
    {/each}

    {#if isLoading}
      <div class="message-wrapper assistant">
        <div class="avatar assistant">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 2L2 7L12 12L22 7L12 2Z"></path>
            <path d="M2 17L12 22L22 17"></path>
            <path d="M2 12L12 17L22 12"></path>
          </svg>
        </div>
        <div class="loading-dots">
          <div class="loading-dot"></div>
          <div class="loading-dot"></div>
          <div class="loading-dot"></div>
        </div>
      </div>
    {/if}
  </div>

  <div class="input-container">
    <form onsubmit={event => { event.preventDefault(); send(); }}>
      <div class="input-wrapper">
        <textarea
          bind:value={input}
          placeholder="Send a message..."
          onkeydown={enterToSend}
          oninput={adjustTextareaHeight}
          disabled={isLoading}
        ></textarea>
        <button type="submit" aria-label="Send message" disabled={!input.trim() || isLoading}>
          <svg class="send-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"></path>
          </svg>
        </button>
      </div>
    </form>
  </div>
</div>
