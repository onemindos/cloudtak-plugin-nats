<template>
    <div class='ag'>
        <!-- ── Top bar ──────────────────────────────────────────────────────── -->
        <div class='ag-bar'>
            <div class='ag-status-row'>
                <div class='ag-dot' :class='bridgeStatus' />
                <span class='ag-bar-label'>
                    {{ bridgeStatusLabel }}
                </span>
                <span v-if='natsServer' class='ag-muted ag-mono'>· {{ natsServer }}</span>
            </div>
            <span class='ag-muted ag-mono'>{{ agentList.length }} agent{{ agentList.length !== 1 ? 's' : '' }}</span>
            <button class='ag-btn' title='Re-discover agents' @click='bridge.discover()'>
                <RefreshCw :size='11' />
                Refresh
            </button>
        </div>

        <!-- ── Body: roster left + chat right ──────────────────────────────── -->
        <div class='ag-body'>
            <!-- Roster -->
            <div class='ag-roster'>
                <div v-if='agentList.length === 0' class='ag-empty'>
                    <span class='ag-empty-icon'>{{ bridgeStatus === 'connected' ? '📡' : '🔌' }}</span>
                    <span>{{ bridgeStatus === 'connected' ? 'No agents discovered' : bridgeStatusLabel }}</span>
                    <span v-if='bridgeStatus === "connected"' class='ag-muted'>Agents publish on agents.hb.*.*.*</span>
                </div>
                <button
                    v-for='a in agentList'
                    :key='a.instanceId'
                    class='ag-card'
                    :class='{ selected: selectedId === a.instanceId }'
                    @click='selectAgent(a.instanceId)'
                >
                    <div class='ag-card-top'>
                        <div class='ag-type-badge' :style='{ background: agentColor(a.agent).bg, color: agentColor(a.agent).fg }'>
                            {{ agentInitials(a.agent) }}
                        </div>
                        <div class='ag-card-info'>
                            <div class='ag-card-name'>{{ a.name || a.agent }}</div>
                            <div class='ag-muted ag-card-sub'>{{ a.agent }}<span v-if='a.owner'> · {{ a.owner }}</span></div>
                        </div>
                        <div class='ag-hb-dot' :class='heartbeatClass(a.instanceId)' />
                    </div>
                    <div v-if='a.description' class='ag-card-desc'>{{ a.description }}</div>
                </button>
            </div>

            <!-- Chat panel -->
            <div class='ag-chat'>
                <template v-if='selectedAgent'>
                    <div class='ag-chat-hd'>
                        <div class='ag-type-badge sm' :style='{ background: agentColor(selectedAgent.agent).bg, color: agentColor(selectedAgent.agent).fg }'>
                            {{ agentInitials(selectedAgent.agent) }}
                        </div>
                        <div>
                            <div class='ag-chat-name'>{{ selectedAgent.name || selectedAgent.agent }}</div>
                            <div class='ag-muted'>{{ selectedAgent.description || selectedAgent.agent }}</div>
                        </div>
                    </div>

                    <div ref='messagesEl' class='ag-messages'>
                        <div v-if='messages.length === 0' class='ag-chat-empty'>
                            <span class='ag-muted'>Send a message to start a conversation</span>
                        </div>
                        <div
                            v-for='(msg, i) in messages'
                            :key='i'
                            class='ag-msg'
                            :class='msg.role'
                        >
                            <div v-if='msg.role === "user"' class='ag-msg-bubble user'>
                                {{ msg.text }}
                            </div>
                            <template v-else-if='msg.role === "assistant"'>
                                <div class='ag-msg-bubble assistant'>
                                    <span v-if='msg.text' class='ag-msg-text'>{{ msg.text }}</span>
                                    <span v-if='!msg.done && !msg.text' class='ag-typing'>
                                        <span /><span /><span />
                                    </span>
                                    <span v-if='!msg.done && msg.text' class='ag-cursor' />
                                </div>
                                <div v-if='msg.toolName' class='ag-tool'>
                                    <Wrench :size='10' />
                                    {{ msg.toolName }}
                                </div>
                                <div v-if='msg.cost' class='ag-cost ag-muted'>${{ msg.cost.toFixed(4) }}</div>
                            </template>
                            <div v-else-if='msg.role === "error"' class='ag-msg-bubble error'>
                                {{ msg.text }}
                            </div>
                            <div v-else-if='msg.role === "status"' class='ag-status-msg ag-muted'>
                                {{ msg.text }}
                            </div>
                        </div>
                    </div>

                    <div class='ag-input-area'>
                        <textarea
                            ref='inputEl'
                            v-model='inputText'
                            class='ag-input'
                            placeholder='Message the agent… (Enter to send, Shift+Enter for newline)'
                            :disabled='isStreaming'
                            rows='3'
                            @keydown='onKeydown'
                        />
                        <div class='ag-input-actions'>
                            <button
                                v-if='isStreaming'
                                class='ag-stop-btn'
                                @click='cancelPrompt'
                            >
                                <StopCircle :size='13' /> Stop
                            </button>
                            <button
                                v-else
                                class='ag-send-btn'
                                :disabled='!inputText.trim()'
                                @click='sendMessage'
                            >
                                <Send :size='13' /> Send
                            </button>
                        </div>
                    </div>
                </template>

                <div v-else class='ag-chat-placeholder'>
                    <span class='ag-ph-icon'>💬</span>
                    <span class='ag-muted'>Select an agent to start a conversation</span>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup lang='ts'>
import { ref, computed, watch, nextTick, onMounted, onUnmounted } from 'vue';
import { RefreshCw, Send, StopCircle, Wrench } from 'lucide-vue-next';
import { useBridge } from '../composables/useBridge';

const bridge = useBridge();
const { status: bridgeStatus, natsServer, agents } = bridge;

const selectedId  = ref<string | null>(null);
const inputText   = ref('');
const messagesEl  = ref<HTMLElement | null>(null);
const inputEl     = ref<HTMLTextAreaElement | null>(null);
const isStreaming  = ref(false);
let currentReqId: string | null = null;

interface ChatMsg {
    role: 'user' | 'assistant' | 'error' | 'status';
    text: string;
    done?: boolean;
    toolName?: string;
    cost?: number;
}

const chatHistory = ref<Map<string, ChatMsg[]>>(new Map());

const agentList     = computed(() => [...agents.value.values()]);
const selectedAgent = computed(() => selectedId.value ? agents.value.get(selectedId.value) ?? null : null);
const messages      = computed(() => selectedId.value ? (chatHistory.value.get(selectedId.value) ?? []) : []);

const lastHeartbeat = ref<Map<string, number>>(new Map());

const bridgeStatusLabel = computed(() => {
    const m: Record<string, string> = {
        connected: 'Connected', connecting: 'Connecting…',
        disconnected: 'Disconnected', error: 'Bridge error',
    };
    return m[bridgeStatus.value] ?? bridgeStatus.value;
});

function heartbeatClass(id: string): string {
    const ts = lastHeartbeat.value.get(id);
    if (!ts) return '';
    return Date.now() - ts < 30_000 ? 'alive' : 'stale';
}

const AGENT_COLORS: Array<{ bg: string; fg: string }> = [
    { bg: 'rgba(99,102,241,0.18)', fg: '#818cf8' },
    { bg: 'rgba(20,184,166,0.18)', fg: '#2dd4bf' },
    { bg: 'rgba(234,179,8,0.18)',  fg: '#fbbf24' },
    { bg: 'rgba(239,68,68,0.18)',  fg: '#f87171' },
    { bg: 'rgba(74,158,255,0.18)', fg: '#4a9eff' },
    { bg: 'rgba(168,85,247,0.18)', fg: '#c084fc' },
];

function agentColor(agent: string): { bg: string; fg: string } {
    let h = 0;
    for (let i = 0; i < agent.length; i++) h = (h * 31 + agent.charCodeAt(i)) & 0xffffffff;
    return AGENT_COLORS[Math.abs(h) % AGENT_COLORS.length];
}

function agentInitials(agent: string): string {
    const parts = agent.replace(/[-_]/g, ' ').split(' ').filter(Boolean);
    if (parts.length >= 2) return (parts[0][0] + parts[1][0]).toUpperCase();
    return agent.slice(0, 2).toUpperCase();
}

function selectAgent(id: string) {
    selectedId.value = id;
    if (!chatHistory.value.has(id)) {
        chatHistory.value = new Map(chatHistory.value).set(id, []);
    }
    nextTick(() => scrollToBottom());
}

function scrollToBottom() {
    if (messagesEl.value) messagesEl.value.scrollTop = messagesEl.value.scrollHeight;
}

function pushMsg(agentId: string, msg: ChatMsg) {
    const next = new Map(chatHistory.value);
    const list = [...(next.get(agentId) ?? [])];
    list.push(msg);
    next.set(agentId, list);
    chatHistory.value = next;
}

function updateLastMsg(agentId: string, patch: Partial<ChatMsg>) {
    const next = new Map(chatHistory.value);
    const list = [...(next.get(agentId) ?? [])];
    if (list.length > 0) {
        list[list.length - 1] = { ...list[list.length - 1], ...patch };
    }
    next.set(agentId, list);
    chatHistory.value = next;
}

function sendMessage() {
    const text = inputText.value.trim();
    const agentId = selectedId.value;
    if (!text || !agentId || isStreaming.value) return;

    inputText.value = '';
    pushMsg(agentId, { role: 'user', text });
    pushMsg(agentId, { role: 'assistant', text: '', done: false });
    isStreaming.value = true;
    nextTick(scrollToBottom);

    currentReqId = bridge.prompt(agentId, text, (ev) => {
        if (ev.kind === 'response') {
            updateLastMsg(agentId, { text: (messages.value[messages.value.length - 1]?.text ?? '') + (ev.text ?? '') });
            nextTick(scrollToBottom);
        } else if (ev.kind === 'tool-use') {
            updateLastMsg(agentId, { toolName: ev.toolName });
        } else if (ev.kind === 'tool-result') {
            updateLastMsg(agentId, { toolName: undefined });
        } else if (ev.kind === 'cost') {
            updateLastMsg(agentId, { cost: ev.totalCostUsd });
        } else if (ev.kind === 'done') {
            updateLastMsg(agentId, { done: true });
            isStreaming.value = false;
            currentReqId = null;
            nextTick(() => inputEl.value?.focus());
        } else if (ev.kind === 'error') {
            updateLastMsg(agentId, { role: 'error', text: ev.message ?? 'Unknown error', done: true });
            isStreaming.value = false;
            currentReqId = null;
        } else if (ev.kind === 'status') {
            pushMsg(agentId, { role: 'status', text: ev.status ?? '' });
            nextTick(scrollToBottom);
        }
    });
}

function cancelPrompt() {
    if (currentReqId) {
        bridge.cancel(currentReqId);
        currentReqId = null;
    }
    if (selectedId.value) updateLastMsg(selectedId.value, { done: true });
    isStreaming.value = false;
}

function onKeydown(e: KeyboardEvent) {
    if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        sendMessage();
    }
}

onMounted(() => bridge.start());
onUnmounted(() => bridge.stop());
</script>

<style scoped>
.ag { display:flex; flex-direction:column; height:100%; overflow:hidden; }

/* Top bar */
.ag-bar { display:flex; align-items:center; gap:10px; padding:6px 12px; flex-shrink:0; border-bottom:1px solid rgba(255,255,255,0.07); background:rgba(255,255,255,0.02); }
.ag-status-row { display:flex; align-items:center; gap:6px; flex:1; }
.ag-dot { width:7px; height:7px; border-radius:50%; background:#555; flex-shrink:0; }
.ag-dot.connected { background:#22c55e; box-shadow:0 0 5px #22c55e88; }
.ag-dot.connecting { background:#f59e0b; animation:blink 1s infinite; }
.ag-dot.error { background:#f85149; }
.ag-bar-label { font-size:11px; font-weight:500; color:rgba(255,255,255,0.6); }
.ag-muted { color:rgba(255,255,255,0.35); font-size:11px; }
.ag-mono { font-family:monospace; }
.ag-btn { display:flex; align-items:center; gap:5px; padding:3px 10px; border-radius:5px; font-size:11px; font-weight:500; cursor:pointer; background:rgba(255,255,255,0.05); border:1px solid rgba(255,255,255,0.1); color:rgba(255,255,255,0.5); transition:all .12s; }
.ag-btn:hover { background:rgba(255,255,255,0.1); color:#e6edf3; }

/* Body */
.ag-body { display:flex; flex:1; overflow:hidden; }

/* Roster */
.ag-roster { width:260px; flex-shrink:0; border-right:1px solid rgba(255,255,255,0.07); overflow-y:auto; display:flex; flex-direction:column; gap:2px; padding:6px; }
.ag-empty { display:flex; flex-direction:column; align-items:center; justify-content:center; gap:6px; height:100%; color:rgba(255,255,255,0.3); font-size:12px; }
.ag-empty-icon { font-size:28px; opacity:0.3; }
.ag-card { display:flex; flex-direction:column; gap:6px; padding:8px 10px; border-radius:8px; border:1px solid transparent; cursor:pointer; text-align:left; background:transparent; transition:all .12s; width:100%; }
.ag-card:hover { background:rgba(255,255,255,0.04); border-color:rgba(255,255,255,0.07); }
.ag-card.selected { background:rgba(74,158,255,0.08); border-color:rgba(74,158,255,0.25); }
.ag-card-top { display:flex; align-items:center; gap:8px; }
.ag-type-badge { width:28px; height:28px; border-radius:6px; display:flex; align-items:center; justify-content:center; font-size:10px; font-weight:700; flex-shrink:0; }
.ag-type-badge.sm { width:24px; height:24px; font-size:9px; }
.ag-card-info { flex:1; min-width:0; }
.ag-card-name { font-size:12px; font-weight:600; color:#e6edf3; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.ag-card-sub { font-size:10px; white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.ag-hb-dot { width:6px; height:6px; border-radius:50%; background:#444; flex-shrink:0; }
.ag-hb-dot.alive { background:#22c55e; box-shadow:0 0 4px #22c55e; }
.ag-hb-dot.stale { background:#f59e0b; }
.ag-card-desc { font-size:10px; color:rgba(255,255,255,0.3); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }

/* Chat area */
.ag-chat { flex:1; display:flex; flex-direction:column; overflow:hidden; }
.ag-chat-hd { display:flex; align-items:center; gap:10px; padding:8px 14px; border-bottom:1px solid rgba(255,255,255,0.07); flex-shrink:0; }
.ag-chat-name { font-size:12px; font-weight:600; color:#e6edf3; }
.ag-messages { flex:1; overflow-y:auto; padding:12px; display:flex; flex-direction:column; gap:10px; }
.ag-chat-empty { display:flex; align-items:center; justify-content:center; height:100%; }
.ag-msg.user { display:flex; justify-content:flex-end; }
.ag-msg.assistant,
.ag-msg.error,
.ag-msg.status { display:flex; flex-direction:column; align-items:flex-start; gap:3px; }
.ag-msg-bubble { max-width:85%; padding:8px 12px; border-radius:10px; font-size:12px; line-height:1.5; white-space:pre-wrap; word-break:break-word; }
.ag-msg-bubble.user { background:rgba(74,158,255,0.15); border:1px solid rgba(74,158,255,0.2); color:#e6edf3; }
.ag-msg-bubble.assistant { background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.07); color:#e6edf3; }
.ag-msg-bubble.error { background:rgba(248,81,73,0.1); border:1px solid rgba(248,81,73,0.2); color:#f85149; }
.ag-status-msg { font-size:10px; font-style:italic; }
.ag-tool { display:flex; align-items:center; gap:4px; font-size:10px; color:rgba(255,255,255,0.4); padding:2px 6px; }
.ag-cost { font-size:10px; padding:2px 6px; font-family:monospace; }
.ag-cursor { display:inline-block; width:2px; height:13px; background:#4a9eff; margin-left:2px; animation:blink 0.8s infinite; vertical-align:middle; }
.ag-typing { display:inline-flex; gap:3px; align-items:center; padding:2px 0; }
.ag-typing span { width:5px; height:5px; border-radius:50%; background:rgba(255,255,255,0.4); animation:bounce 1.2s infinite; }
.ag-typing span:nth-child(2) { animation-delay:.2s; }
.ag-typing span:nth-child(3) { animation-delay:.4s; }
@keyframes bounce { 0%,80%,100%{transform:translateY(0)} 40%{transform:translateY(-4px)} }
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:0} }

/* Input */
.ag-input-area { flex-shrink:0; border-top:1px solid rgba(255,255,255,0.07); padding:10px; display:flex; gap:8px; align-items:flex-end; }
.ag-input { flex:1; resize:none; background:rgba(255,255,255,0.04); border:1px solid rgba(255,255,255,0.1); border-radius:8px; color:#e6edf3; font-size:12px; padding:8px 10px; outline:none; font-family:inherit; line-height:1.5; transition:border-color .12s; }
.ag-input:focus { border-color:rgba(74,158,255,0.4); }
.ag-input:disabled { opacity:0.5; cursor:not-allowed; }
.ag-send-btn { display:flex; align-items:center; gap:5px; padding:6px 14px; border-radius:6px; font-size:12px; font-weight:600; cursor:pointer; background:#4a9eff; color:#000; border:none; transition:opacity .15s; white-space:nowrap; }
.ag-send-btn:disabled { opacity:0.35; cursor:not-allowed; }
.ag-send-btn:not(:disabled):hover { opacity:0.85; }
.ag-stop-btn { display:flex; align-items:center; gap:5px; padding:6px 14px; border-radius:6px; font-size:12px; font-weight:600; cursor:pointer; background:rgba(248,81,73,0.15); color:#f85149; border:1px solid rgba(248,81,73,0.25); transition:all .12s; white-space:nowrap; }
.ag-stop-btn:hover { background:rgba(248,81,73,0.25); }

/* Empty placeholder */
.ag-chat-placeholder { display:flex; flex-direction:column; align-items:center; justify-content:center; height:100%; gap:10px; }
.ag-ph-icon { font-size:36px; opacity:0.2; }
</style>
