<script lang="ts">
  import { onMount } from 'svelte';
  import { marked } from 'marked';

  // --- state --------------------------------------------------------------
  let messages = $state<{ role: string; content: string }[]>([]);
  let input = $state('');
  let isLoading = $state(false);
  let messagesContainer: HTMLDivElement;

  // Configuration variables, set once onMount.
  let apiKey: string = '';
  let model = $state('gpt-4o-mini');
  let endpoint: string = '/api/chat/completions';
  let toolIds: string[] = [];
  let title = $state('');
  let prompts = $state<string[]>([]);

  const defaultPrompts = [
    'What is the cost of running an e2-medium VM in me-central2?',
    'Estimate monthly cost for Cloud SQL in me-central2',
    'What GCP services are available in me-central2?'
  ];

  // Configure marked for secure rendering
  marked.setOptions({ breaks: true, gfm: true });

  function renderMarkdown(content: string): string {
    return marked.parse(content) as string;
  }

  function displayTitle(): string {
    return title || 'GCP Cost Assistant';
  }

  // --- initialize from query-string --------------------------------------
  onMount(() => {
    try {
      const qs = new URLSearchParams(window.location.search);
      if (qs.get('api_key') !== null) apiKey = qs.get('api_key')!;
      if (qs.get('model') !== null) model = qs.get('model')!;
      if (qs.get('endpoint') !== null) endpoint = qs.get('endpoint')!;
      if (qs.get('tool_ids') !== null) toolIds = qs.get('tool_ids')!.split(',').filter(Boolean);
      if (qs.get('title') !== null) title = qs.get('title')!;
      if (qs.get('prompts') !== null) prompts = qs.get('prompts')!.split('|').filter(Boolean);
    } catch (e) {
      console.error('Error processing query parameters:', e);
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
  function clearChat() {
    messages = [];
    input = '';
  }

  function sendPrompt(text: string) {
    input = text;
    send();
  }

  async function send() {
    const text = input.trim();
    if (!text || isLoading) return;

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
          messages: messages.map(m => ({ role: m.role, content: m.content })),
          stream: true,
          ...(toolIds.length > 0 && { tool_ids: toolIds })
        })
      });

      if (!res.ok) {
        messages = [...messages, { role: 'assistant', content: '\u26a0\ufe0f Error retrieving response' }];
        return;
      }

      const reader = res.body?.getReader();
      if (!reader) {
        messages = [...messages, { role: 'assistant', content: '\u26a0\ufe0f Error retrieving response' }];
        return;
      }

      const decoder = new TextDecoder();
      let assistantContent = '';
      messages = [...messages, { role: 'assistant', content: '' }];
      isLoading = false;

      let buffer = '';
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() ?? '';

        for (const line of lines) {
          if (!line.startsWith('data: ')) continue;
          const data = line.slice(6).trim();
          if (data === '[DONE]') continue;

          try {
            const parsed = JSON.parse(data);
            const delta = parsed?.choices?.[0]?.delta?.content;
            if (delta) {
              assistantContent += delta;
              messages = [...messages.slice(0, -1), { role: 'assistant', content: assistantContent }];
            }
          } catch {
            // skip non-JSON lines
          }
        }
      }

      if (!assistantContent) {
        messages = [...messages.slice(0, -1), { role: 'assistant', content: '\u26a0\ufe0f Error retrieving response' }];
      }
    } catch (err) {
      messages = [...messages, { role: 'assistant', content: '\u26a0\ufe0f Network error' }];
    } finally {
      isLoading = false;
    }
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
    padding: 0.875rem 1.25rem;
    background: #ffffff;
    border-bottom: 1px solid #e5e5e5;
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .header-icon {
    width: 28px;
    height: 28px;
    background: #4285f4;
    border-radius: 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
    font-weight: 600;
    font-size: 16px;
  }

  .header-title {
    font-size: 0.9375rem;
    font-weight: 600;
    color: #202123;
    flex: 1;
  }

  .header-actions {
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }

  .header-btn {
    padding: 0.25rem;
    border: none;
    border-radius: 4px;
    background: transparent;
    color: #8e8ea0;
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    transition: all 0.15s ease;
  }

  .header-btn:hover {
    color: #202123;
    background: #f0f0f0;
  }

  /* Welcome screen */
  .welcome {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 2rem 1.5rem;
    text-align: center;
    gap: 1rem;
  }

  .welcome-icon {
    width: 48px;
    height: 48px;
    background: #4285f4;
    border-radius: 12px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: white;
  }

  .welcome-title {
    font-size: 1.125rem;
    font-weight: 600;
    color: #202123;
    margin: 0;
  }

  .welcome-subtitle {
    font-size: 0.8125rem;
    color: #8e8ea0;
    margin: 0;
    line-height: 1.5;
  }

  .suggested-prompts {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    width: 100%;
    max-width: 320px;
    margin-top: 0.5rem;
  }

  .prompt-btn {
    padding: 0.625rem 0.875rem;
    border: 1px solid #e5e5e5;
    border-radius: 8px;
    background: #ffffff;
    color: #353740;
    font-size: 0.8125rem;
    font-family: inherit;
    text-align: left;
    cursor: pointer;
    transition: all 0.15s ease;
    line-height: 1.4;
    width: 100%;
    display: block;
  }

  .prompt-btn:hover {
    border-color: #4285f4;
    background: #f8f9ff;
  }

  /* Messages area */
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
    width: 28px;
    height: 28px;
    border-radius: 6px;
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
    background: #4285f4;
  }

  .message {
    flex: 1;
    line-height: 1.75;
    font-size: 0.875rem;
    color: #202123;
    word-wrap: break-word;
    animation: slideIn 0.3s ease-out;
  }

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
    color: #4285f4;
    text-decoration: underline;
  }

  :global(.markdown-content a:hover) {
    color: #3367d6;
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
    border-radius: 8px;
    padding: 0.75rem 0.75rem 0.75rem 1rem;
    transition: border-color 0.2s ease;
  }

  .input-wrapper:focus-within {
    border-color: #4285f4;
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

  .send-btn {
    padding: 0.375rem;
    border: none;
    border-radius: 6px;
    background: #4285f4;
    color: #ffffff;
    cursor: pointer;
    transition: all 0.15s ease;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 32px;
    height: 32px;
    flex-shrink: 0;
  }

  .send-btn:hover:not(:disabled) {
    background: #3367d6;
  }

  .send-btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  .send-icon {
    width: 16px;
    height: 16px;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  @keyframes slideIn {
    from { transform: translateY(5px); opacity: 0; }
    to { transform: translateY(0); opacity: 1; }
  }

  @keyframes bounce {
    0%, 80%, 100% { transform: scale(0); }
    40% { transform: scale(1); }
  }
</style>

<div class="wrapper">
  <div class="header">
    <div class="header-icon">
      <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
        <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
      </svg>
    </div>
    <div class="header-title">{displayTitle()}</div>
    <div class="header-actions">
      {#if messages.length > 0}
        <button class="header-btn" onclick={clearChat} title="New chat">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 5v14M5 12h14"></path>
          </svg>
        </button>
      {/if}
    </div>
  </div>

  {#if messages.length === 0 && !isLoading}
    <div class="welcome">
      <div class="welcome-icon">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
        </svg>
      </div>
      <h2 class="welcome-title">How can I help?</h2>
      <p class="welcome-subtitle">Ask questions about GCP costs, pricing, and service availability across regions.</p>
      <div class="suggested-prompts">
        {#each prompts.length > 0 ? prompts : defaultPrompts as prompt}
          <button class="prompt-btn" onclick={() => sendPrompt(prompt)}>
            {prompt}
          </button>
        {/each}
      </div>
    </div>
  {:else}
    <div class="messages" bind:this={messagesContainer}>
      {#each messages as message}
        <div class="message-wrapper {message.role}">
          <div class="avatar {message.role}">
            {#if message.role === 'user'}
              <svg width="14" height="14" viewBox="0 0 24 24" fill="currentColor">
                <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
                <circle cx="12" cy="7" r="4"></circle>
              </svg>
            {:else}
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
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
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M21 15a2 2 0 0 1-2 2H7l-4 4V5a2 2 0 0 1 2-2h14a2 2 0 0 1 2 2z"></path>
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
  {/if}

  <div class="input-container">
    <form onsubmit={event => { event.preventDefault(); send(); }}>
      <div class="input-wrapper">
        <textarea
          bind:value={input}
          placeholder="Ask about GCP costs and pricing..."
          onkeydown={enterToSend}
          oninput={adjustTextareaHeight}
          disabled={isLoading}
        ></textarea>
        <button type="submit" class="send-btn" aria-label="Send message" disabled={!input.trim() || isLoading}>
          <svg class="send-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"></path>
          </svg>
        </button>
      </div>
    </form>
  </div>
</div>
