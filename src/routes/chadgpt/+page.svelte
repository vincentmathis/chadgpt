<script lang="ts">
type Message = { role: 'user' | 'assistant'; content: string };
type Conversation = { id: string; title: string; messages: Message[] };

const STORAGE_KEY = 'chadgpt-conversations';

let conversations = $state<Conversation[]>([]);
let activeId = $state<string | null>(null);
let input = $state('');
let streaming = $state(false);
let abortController: AbortController | null = null;
let sidebarOpen = $state(false);
let persona = $state<'chadgpt' | 'winkler'>('chadgpt');

let messagesContainer: HTMLDivElement;

let active = $derived(conversations.find((c) => c.id === activeId) ?? null);

// Load from localStorage on mount
if (typeof window !== 'undefined') {
    try {
        const saved = localStorage.getItem(STORAGE_KEY);
        if (saved) {
            conversations = JSON.parse(saved);
        }
        const savedActive = localStorage.getItem(STORAGE_KEY + '-active');
        if (savedActive && conversations.some((c) => c.id === savedActive)) {
            activeId = savedActive;
        } else if (conversations.length > 0) {
            activeId = conversations[0].id;
        }
    } catch {
        /* ignore corrupt data */
    }
}

// Save to localStorage whenever conversations change
$effect(() => {
    if (typeof window !== 'undefined' && conversations.length >= 0) {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(conversations));
    }
});

$effect(() => {
    if (typeof window !== 'undefined' && activeId) {
        localStorage.setItem(STORAGE_KEY + '-active', activeId);
    }
});

function newChat() {
    const id = crypto.randomUUID();
    conversations = [{ id, title: 'New Chat', messages: [] }, ...conversations];
    activeId = id;
}

function selectChat(id: string) {
    activeId = id;
    sidebarOpen = false;
}

function deleteChat(id: string) {
    conversations = conversations.filter((c) => c.id !== id);
    if (activeId === id) {
        activeId = conversations[0]?.id ?? null;
    }
}

function scrollToBottom() {
    requestAnimationFrame(() => {
        if (messagesContainer) {
            messagesContainer.scrollTop = messagesContainer.scrollHeight;
        }
    });
}

async function send() {
    const text = input.trim();
    if (!text || streaming) return;

    if (!active) {
        newChat();
    }

    input = '';

    active.messages = [...active.messages, { role: 'user', content: text }];

    // Auto-title from first user message
    if (active.messages.filter((m) => m.role === 'user').length === 1) {
        active.title = text.length > 40 ? text.slice(0, 40) + '...' : text;
    }

    // Add empty assistant message
    active.messages = [...active.messages, { role: 'assistant', content: '' }];
    streaming = true;
    scrollToBottom();

    abortController = new AbortController();

    try {
        const res = await fetch('/chadgpt/chat/', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
                messages: active.messages.slice(0, -1).map((m) => ({
                    role: m.role,
                    content: m.content,
                })),
                persona,
            }),
            signal: abortController.signal,
        });

        if (!res.ok) {
            throw new Error(`HTTP ${res.status}`);
        }

        const reader = res.body!.getReader();
        const decoder = new TextDecoder();

        while (true) {
            const { done, value } = await reader.read();
            if (done) break;

            const chunk = decoder.decode(value, { stream: true });
            const lastMsg = active.messages[active.messages.length - 1];
            if (lastMsg.role === 'assistant') {
                lastMsg.content += chunk;
                // Trigger reactivity
                active.messages = [...active.messages];
            }
            scrollToBottom();
        }
    } catch (err: unknown) {
        if (err instanceof DOMException && err.name === 'AbortError') {
            // User cancelled
        } else {
            const lastMsg = active.messages[active.messages.length - 1];
            if (lastMsg.role === 'assistant') {
                lastMsg.content += '\n\n[Connection error — try again]';
                active.messages = [...active.messages];
            }
        }
    } finally {
        streaming = false;
        abortController = null;
    }
}

function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        send();
    }
}

function stop() {
    abortController?.abort();
}

// Auto-select first conversation
$effect(() => {
    if (!activeId && conversations.length > 0) {
        activeId = conversations[0].id;
    }
});
</script>

<div class="app">
    <!-- Sidebar backdrop -->
    <div
        class="sidebar-backdrop"
        class:visible={sidebarOpen}
        onclick={() => (sidebarOpen = false)}
        role="button"
        tabindex="-1"
    ></div>

    <!-- Sidebar -->
    <aside class="sidebar" class:open={sidebarOpen}>
        <button class="new-chat" onclick={newChat}>
            <svg
                width="16"
                height="16"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
            >
                <path d="M12 5v14M5 12h14" />
            </svg>
            New Chat
        </button>

        <div class="chat-list">
            {#each conversations as conv (conv.id)}
                <div
                    class="chat-item"
                    class:active={conv.id === activeId}
                    onclick={() => selectChat(conv.id)}
                    role="button"
                    tabindex="0"
                    onkeydown={(e) => { if (e.key === 'Enter') selectChat(conv.id); }}
                >
                    <span class="chat-title">{conv.title}</span>
                    <button
                        class="delete-btn"
                        onclick={(e) => { e.stopPropagation(); deleteChat(conv.id); }}
                        aria-label="Delete chat"
                    >
                        <svg
                            width="14"
                            height="14"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="2"
                        >
                            <path d="M18 6L6 18M6 6l12 12" />
                        </svg>
                    </button>
                </div>
            {/each}
        </div>

        <div class="sidebar-footer">
            <div class="persona-selector">
                <button
                    class="persona-btn"
                    class:active={persona === 'chadgpt'}
                    onclick={() => persona = 'chadgpt'}
                >
                    ChadGPT
                </button>
                <button
                    class="persona-btn"
                    class:active={persona === 'winkler'}
                    onclick={() => persona = 'winkler'}
                >
                    Winkler
                </button>
            </div>
        </div>
    </aside>

    <!-- Main -->
    <main class="main">
        <button
            class="hamburger"
            onclick={() => (sidebarOpen = !sidebarOpen)}
            aria-label="Toggle chats"
        >
            <svg
                width="20"
                height="20"
                viewBox="0 0 24 24"
                fill="none"
                stroke="currentColor"
                stroke-width="2"
            >
                <path d="M3 12h18M3 6h18M3 18h18" />
            </svg>
        </button>

        {#if active && active.messages.length > 0}
            <div class="messages" bind:this={messagesContainer}>
                {#each active.messages as msg}
                    <div
                        class="message"
                        class:user={msg.role === 'user'}
                        class:assistant={msg.role === 'assistant'}
                    >
                        <div class="avatar">
                            {#if msg.role === 'user'}
                                <span class="avatar-user">U</span>
                            {:else}
                                <span class="avatar-bot">C</span>
                            {/if}
                        </div>
                        <div class="bubble">
                            {#if msg.role === 'assistant' && streaming && msg === active.messages[active.messages.length - 1] && !msg.content}
                                <span class="cursor"></span>
                            {:else}
                                {msg.content}
                                {#if msg.role === 'assistant' && streaming && msg === active.messages[active.messages.length - 1]}
                                    <span class="cursor"></span>
                                {/if}
                            {/if}
                        </div>
                    </div>
                {/each}
            </div>
        {/if}

        {#if !active || active.messages.length === 0}
            <div class="empty-state">
                <div class="empty-icon">💬</div>
                <h2>{active ? "What's on your mind?" : 'ChadGPT'}</h2>
            </div>
        {/if}

        <div class="input-area" class:centered={!active || active.messages.length === 0}>
            <div class="input-wrapper">
                <textarea
                    bind:value={input}
                    onkeydown={handleKeydown}
                    placeholder="Message ChadGPT..."
                    rows="1"
                    disabled={streaming}
                ></textarea>
                {#if streaming}
                    <button class="stop-btn" onclick={stop} aria-label="Stop">
                        <svg width="18" height="18" viewBox="0 0 24 24" fill="currentColor">
                            <rect x="6" y="6" width="12" height="12" rx="2" />
                        </svg>
                    </button>
                {:else}
                    <button
                        class="send-btn"
                        onclick={send}
                        disabled={!input.trim()}
                        aria-label="Send"
                    >
                        <svg
                            width="18"
                            height="18"
                            viewBox="0 0 24 24"
                            fill="none"
                            stroke="currentColor"
                            stroke-width="2"
                        >
                            <path d="M22 2L11 13M22 2l-7 20-4-9-9-4z" />
                        </svg>
                    </button>
                {/if}
            </div>
            <p class="disclaimer">ChadGPT can make mistakes. Verify important info.</p>
        </div>
    </main>
</div>

<style>
:global(body) {
    background: #0d0d0d;
    overflow: hidden;
    height: 100vh;
    height: 100dvh;
    user-select: text;
}

.app {
    display: flex;
    height: 100vh;
    height: 100dvh;
    font-family: "Inter", system-ui, sans-serif;
    color: #e5e5e5;
}

/* ── Sidebar ── */
.sidebar {
    width: 260px;
    min-width: 260px;
    background: #111;
    border-right: 1px solid #1f1f1f;
    display: flex;
    flex-direction: column;
    padding: 12px;
    gap: 8px;
}

.new-chat {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border: 1px solid #2a2a2a;
    border-radius: 8px;
    background: transparent;
    color: #e5e5e5;
    font-size: 14px;
    cursor: pointer;
    transition: background 0.15s;
}
.new-chat:hover {
    background: #1a1a1a;
}

.chat-list {
    flex: 1;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 2px;
}

.chat-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 10px;
    border-radius: 6px;
    background: transparent;
    color: #a0a0a0;
    font-size: 13px;
    border: none;
    cursor: pointer;
    text-align: left;
    transition: background 0.15s;
}
.chat-item:hover {
    background: #1a1a1a;
    color: #e5e5e5;
}
.chat-item.active {
    background: #1a1a1a;
    color: #e5e5e5;
}

.chat-title {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    flex: 1;
}

.delete-btn {
    opacity: 0;
    background: none;
    border: none;
    color: #666;
    cursor: pointer;
    padding: 2px;
    border-radius: 4px;
    display: flex;
    transition:
        opacity 0.15s,
        color 0.15s;
}
.chat-item:hover .delete-btn {
    opacity: 1;
}
.delete-btn:hover {
    color: #ef4444;
}

.sidebar-footer {
    padding: 12px 4px 4px;
    border-top: 1px solid #1f1f1f;
}

.persona-selector {
    display: flex;
    gap: 4px;
}

.persona-btn {
    flex: 1;
    padding: 6px 8px;
    border: 1px solid #2a2a2a;
    border-radius: 6px;
    background: transparent;
    color: #666;
    font-size: 12px;
    font-family: "JetBrains Mono", monospace;
    cursor: pointer;
    transition: all 0.15s;
}

.persona-btn:hover {
    color: #e5e5e5;
    border-color: #444;
}

.persona-btn.active {
    background: #1a1a1a;
    color: #e5e5e5;
    border-color: #444;
}

/* ── Main ── */
.main {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-width: 0;
    position: relative;
}

.messages {
    flex: 1;
    overflow-y: auto;
    padding: 24px 0;
}

.message {
    display: flex;
    gap: 12px;
    padding: 12px 24px;
    max-width: 800px;
    margin: 0 auto;
    width: 100%;
    box-sizing: border-box;
}

.message.user {
    flex-direction: row-reverse;
}

.avatar {
    flex-shrink: 0;
    width: 32px;
    height: 32px;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 13px;
    font-weight: 600;
}

.avatar-user {
    background: #3b82f6;
    color: white;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
}

.avatar-bot {
    background: #1a1a1a;
    border: 1px solid #333;
    color: #10b981;
    width: 100%;
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
}

.bubble {
    padding: 10px 14px;
    border-radius: 12px;
    font-size: 14px;
    line-height: 1.6;
    max-width: 100%;
    white-space: pre-wrap;
    word-break: break-word;
}

.message.assistant .bubble {
    background: #1a1a1a;
    border: 1px solid #222;
    color: #e5e5e5;
}

.message.user .bubble {
    background: #1e3a5f;
    border: 1px solid #2563eb33;
    color: #e5e5e5;
}

.cursor {
    display: inline-block;
    width: 2px;
    height: 1em;
    background: #10b981;
    margin-left: 1px;
    vertical-align: text-bottom;
    animation: blink 0.8s step-end infinite;
}

@keyframes blink {
    50% {
        opacity: 0;
    }
}

/* ── Empty state ── */
.empty-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 8px;
    color: #555;
    padding-bottom: 2rem;
}
.empty-icon {
    font-size: 48px;
    margin-bottom: 8px;
}
.empty-state h2 {
    font-size: 24px;
    font-weight: 500;
    color: #888;
    margin: 0;
}
.empty-state p {
    font-size: 14px;
    color: #555;
    margin: 0;
}

/* ── Input ── */
.input-area {
    padding: 0 24px 20px;
    max-width: 800px;
    margin: 0 auto;
    width: 100%;
    box-sizing: border-box;
    transition: all 0.4s cubic-bezier(0.16, 1, 0.3, 1);
}

.input-area.centered {
    width: 100%;
    max-width: 500px;
    margin: 0 auto;
    padding-bottom: 10vh;
}

.input-area.centered .disclaimer {
    margin-top: 1.5rem;
}

.input-wrapper {
    display: flex;
    align-items: center;
    gap: 8px;
    background: #1a1a1a;
    border: 1px solid #2a2a2a;
    border-radius: 12px;
    padding: 10px 12px;
    transition: border-color 0.2s;
}
.input-wrapper:focus-within {
    border-color: #444;
}

textarea {
    flex: 1;
    background: transparent;
    border: none;
    color: #e5e5e5;
    font-size: 14px;
    font-family: inherit;
    resize: none;
    outline: none;
    line-height: 1.5;
    max-height: 150px;
}
textarea::placeholder {
    color: #555;
}

.send-btn,
.stop-btn {
    flex-shrink: 0;
    width: 34px;
    height: 34px;
    border-radius: 8px;
    border: none;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
    transition: background 0.15s;
}

.send-btn {
    background: #10b981;
    color: white;
}
.send-btn:hover:not(:disabled) {
    background: #059669;
}
.send-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
}

.stop-btn {
    background: #ef4444;
    color: white;
}
.stop-btn:hover {
    background: #dc2626;
}

.disclaimer {
    text-align: center;
    font-size: 11px;
    color: #444;
    margin-top: 8px;
}

/* ── Scrollbar ── */
.messages::-webkit-scrollbar,
.chat-list::-webkit-scrollbar {
    width: 6px;
}
.messages::-webkit-scrollbar-track,
.chat-list::-webkit-scrollbar-track {
    background: transparent;
}
.messages::-webkit-scrollbar-thumb,
.chat-list::-webkit-scrollbar-thumb {
    background: #333;
    border-radius: 3px;
}

/* ── Hamburger ── */
.hamburger {
    display: none;
    background: none;
    border: none;
    color: #e5e5e5;
    cursor: pointer;
    padding: 8px;
    border-radius: 6px;
    flex-shrink: 0;
}
.hamburger:hover {
    background: #1a1a1a;
}

/* ── Mobile ── */
@media (max-width: 700px) {
    .hamburger {
        display: flex;
        position: absolute;
        top: 12px;
        left: 12px;
        z-index: 30;
    }

    .sidebar {
        position: fixed;
        top: 0;
        left: 0;
        bottom: 0;
        z-index: 35;
        transform: translateX(-100%);
        transition: transform 0.25s ease;
    }
    .sidebar.open {
        transform: translateX(0);
    }

    .sidebar-backdrop {
        display: none;
    }
    .sidebar-backdrop.visible {
        display: block;
        position: fixed;
        inset: 0;
        background: rgba(0, 0, 0, 0.5);
        z-index: 30;
    }

    .main {
        width: 100%;
    }

    .input-area {
        padding: 0 12px 12px;
    }
    .message {
        padding: 8px 12px;
    }
}
</style>
