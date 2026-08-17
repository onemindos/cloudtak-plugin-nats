<template>
    <div class='security-root'>
        <div class='sec-toolbar'>
            <Shield
                :size='14'
                class='sec-icon'
            />
            <span class='sec-title'>Security</span>
            <span class='sec-subtitle'>$SYS audit log</span>
            <div class='sec-spacer' />
            <span class='sec-count'>{{ events.length }} events</span>
            <button
                class='sec-btn'
                @click='clear'
            >
                <Trash2 :size='13' />
            </button>
        </div>

        <div
            v-if='!nc'
            class='sec-empty'
        >
            <Shield
                :size='28'
                style='opacity:0.3'
            />
            <p>Not connected to NATS.</p>
        </div>

        <div
            v-else
            class='sec-events'
        >
            <div
                v-for='ev in events'
                :key='ev.id'
                class='sec-event'
                :class='ev.type'
            >
                <span
                    class='sec-ev-badge'
                    :class='ev.type'
                >{{ ev.type.toUpperCase() }}</span>
                <span class='sec-ev-ts'>{{ formatTs(ev.ts) }}</span>
                <span class='sec-ev-client'>{{ ev.clientId ?? '—' }}</span>
                <span
                    class='sec-ev-subject'
                    :title='ev.subject'
                >{{ ev.subject }}</span>
                <span
                    class='sec-ev-detail'
                    :title='ev.detail'
                >{{ ev.detail }}</span>
            </div>
            <div
                v-if='!events.length'
                class='sec-empty-inline'
            >
                Listening on $SYS… no events yet.
            </div>
        </div>
    </div>
</template>

<script setup lang='ts'>
import { ref, onMounted, onUnmounted } from 'vue';
import { Shield, Trash2 }  from 'lucide-vue-next';
import type { Subscription } from 'nats.ws';
import { useNatsStore }      from '../stores/nats.store';

interface SecEvent {
    id: string;
    type: 'auth_err' | 'connect' | 'disconnect';
    ts: number;
    subject: string;
    clientId?: string;
    detail: string;
}

const { nc, sc } = useNatsStore();

const events = ref<SecEvent[]>([]);
const subs   = ref<Subscription[]>([]);
let seq = 0;

const SYS_SUBJECTS: Array<{ pattern: string; type: SecEvent['type'] }> = [
    { pattern: '$SYS.SERVER.*.CLIENT.AUTH.ERR', type: 'auth_err'    },
    { pattern: '$SYS.SERVER.*.CLIENT.CONNECT',  type: 'connect'     },
    { pattern: '$SYS.SERVER.*.CLIENT.DISCONNECT',type: 'disconnect' },
];

function startListening() {
    if (!nc.value) return;
    for (const { pattern, type } of SYS_SUBJECTS) {
        const sub = nc.value.subscribe(pattern);
        subs.value.push(sub);
        void (async () => {
            for await (const msg of sub) {
                let data: Record<string, unknown> = {};
                try { data = JSON.parse(sc.decode(msg.data)); } catch { /* ignore */ }
                events.value = [{
                    id:       String(++seq),
                    type,
                    ts:       Date.now(),
                    subject:  msg.subject,
                    clientId: (data?.client as Record<string, unknown>)?.id as string ?? data?.cid as string,
                    detail:   data?.reason as string ?? (data?.client as Record<string, unknown>)?.name as string ?? data?.error as string ?? '',
                }, ...events.value].slice(0, 500);
            }
        })();
    }
}

function clear() { events.value = []; }

function stopListening() {
    subs.value.forEach(s => s.unsubscribe());
    subs.value = [];
}

function formatTs(ts: number) {
    return new Date(ts).toISOString().slice(11, 23);
}

onMounted(startListening);
onUnmounted(stopListening);
</script>

<style scoped>
.security-root {
    display: flex; flex-direction: column; height: 100%; overflow: hidden;
    background: var(--cloudtak-panel-bg, rgba(13,17,23,0.98));
    color: #e6edf3; font-size: 12px;
}
.sec-toolbar {
    display: flex; align-items: center; gap: 7px;
    padding: 10px 12px; border-bottom: 1px solid rgba(255,255,255,0.07);
    flex-shrink: 0;
}
.sec-icon { color: #f85149; }
.sec-title { font-weight: 700; font-size: 13px; }
.sec-subtitle { font-size: 11px; color: rgba(255,255,255,0.35); }
.sec-spacer { flex: 1; }
.sec-count { font-size: 11px; color: rgba(255,255,255,0.3); }
.sec-btn {
    background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.1);
    border-radius: 5px; color: rgba(255,255,255,0.5); padding: 3px 7px;
    cursor: pointer; display: flex; align-items: center;
}
.sec-btn:hover { color: #f85149; }

.sec-empty {
    flex: 1; display: flex; flex-direction: column;
    align-items: center; justify-content: center;
    gap: 10px; opacity: 0.4; color: rgba(255,255,255,0.5); text-align: center;
}
.sec-empty-inline { padding: 24px; text-align: center; color: rgba(255,255,255,0.3); }

.sec-events { flex: 1; overflow-y: auto; }
.sec-event {
    display: flex; align-items: center; gap: 8px;
    padding: 5px 12px; border-bottom: 1px solid rgba(255,255,255,0.04);
    font-size: 11px; font-family: monospace;
}
.sec-event:hover { background: rgba(255,255,255,0.03); }

.sec-ev-badge {
    flex-shrink: 0; font-size: 9px; font-weight: 700; padding: 1px 5px; border-radius: 3px;
    min-width: 68px; text-align: center;
}
.sec-ev-badge.auth_err   { background: rgba(248,81,73,0.15); color: #f85149; }
.sec-ev-badge.connect    { background: rgba(34,197,94,0.12); color: #22c55e; }
.sec-ev-badge.disconnect { background: rgba(245,158,11,0.12); color: #f59e0b; }

.sec-ev-ts       { color: rgba(255,255,255,0.2); min-width: 82px; flex-shrink: 0; }
.sec-ev-client   { color: rgba(255,255,255,0.4); min-width: 60px; flex-shrink: 0; }
.sec-ev-subject  { color: #60a5fa; flex: 1; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.sec-ev-detail   { color: rgba(255,255,255,0.35); max-width: 200px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
</style>
