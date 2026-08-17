<template>
    <div class="nats-console">
        <!-- ── Tab bar ──────────────────────────────────────────────────────── -->
        <div class="nc-tabs">
            <button
                v-for="tab in TABS"
                :key="tab.id"
                class="nc-tab"
                :class="{ active: activeTab === tab.id }"
                @click="activeTab = tab.id"
            >
                <component :is="tab.icon" :size="12" />
                {{ tab.label }}
            </button>
            <div class="nc-tabs-spacer" />
            <!-- NATS connection status pill -->
            <div class="nc-status" :class="status">
                <div class="nc-status-dot" />
                <span>{{ statusLabel }}</span>
                <span v-if="rtt" class="nc-rtt">{{ rtt }}ms</span>
            </div>
        </div>

        <!-- ── Tab content ──────────────────────────────────────────────────── -->
        <div class="nc-body">
            <WireTab     v-if="activeTab === 'wire'" />
            <SecurityTab v-if="activeTab === 'security'" />
            <NatsSettings v-if="activeTab === 'settings'" />
            <!-- Placeholder stubs for upcoming tabs -->
            <div v-if="activeTab === 'topology'"    class="nc-stub">Topology — coming soon</div>
            <div v-if="activeTab === 'connections'" class="nc-stub">Connections — coming soon</div>
            <div v-if="activeTab === 'streams'"     class="nc-stub">Streams — coming soon</div>
            <div v-if="activeTab === 'kv'"          class="nc-stub">Key-Value — coming soon</div>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue';
import {
    Radio, Shield, Network, Link2, Layers, Database, Settings
} from 'lucide-vue-next';
import { useNatsStore } from '../stores/nats.store';
import WireTab      from './WireTab.vue';
import SecurityTab  from './SecurityTab.vue';
import NatsSettings from './NatsSettings.vue';

const { status, rtt } = useNatsStore();

const TABS = [
    { id: 'wire',        label: 'Wire',        icon: Radio    },
    { id: 'security',    label: 'Security',    icon: Shield   },
    { id: 'topology',    label: 'Topology',    icon: Network  },
    { id: 'connections', label: 'Connections', icon: Link2    },
    { id: 'streams',     label: 'Streams',     icon: Layers   },
    { id: 'kv',          label: 'Key-Value',   icon: Database },
    { id: 'settings',    label: 'Settings',    icon: Settings },
] as const;

type TabId = typeof TABS[number]['id'];
const activeTab = ref<TabId>('wire');

const statusLabel = computed(() => {
    const map: Record<string, string> = {
        connected:    'connected',
        connecting:   'connecting…',
        disconnected: 'disconnected',
        error:        'error',
    };
    return map[status.value] ?? status.value;
});
</script>

<style scoped>
.nats-console {
    display: flex;
    flex-direction: column;
    height: 100%;
    overflow: hidden;
    background: var(--cloudtak-panel-bg, rgba(13,17,23,0.98));
    color: #e6edf3;
    font-size: 12px;
}

/* Tab bar */
.nc-tabs {
    display: flex;
    align-items: center;
    flex-shrink: 0;
    border-bottom: 1px solid rgba(255,255,255,0.07);
    padding: 0 8px;
    gap: 1px;
    min-height: 36px;
}
.nc-tab {
    display: flex; align-items: center; gap: 5px;
    padding: 6px 10px;
    background: none; border: none; border-bottom: 2px solid transparent;
    color: rgba(255,255,255,0.4);
    font-size: 11px; font-weight: 500;
    cursor: pointer; white-space: nowrap;
    transition: color .12s, border-color .12s;
}
.nc-tab:hover { color: rgba(255,255,255,0.7); }
.nc-tab.active { color: #4a9eff; border-bottom-color: #4a9eff; }

.nc-tabs-spacer { flex: 1; }

/* Status pill */
.nc-status {
    display: flex; align-items: center; gap: 5px;
    padding: 3px 9px;
    border-radius: 12px;
    font-size: 11px;
    background: rgba(255,255,255,0.05);
    border: 1px solid rgba(255,255,255,0.08);
    color: rgba(255,255,255,0.4);
}
.nc-status-dot {
    width: 6px; height: 6px;
    border-radius: 50%;
    background: rgba(255,255,255,0.2);
}
.nc-status.connected .nc-status-dot    { background: #22c55e; box-shadow: 0 0 6px #22c55e88; }
.nc-status.connected                   { color: #22c55e; border-color: rgba(34,197,94,0.25); }
.nc-status.connecting .nc-status-dot   { background: #f59e0b; animation: blink 1s infinite; }
.nc-status.connecting                  { color: #f59e0b; }
.nc-status.error .nc-status-dot        { background: #f85149; }
.nc-status.error                       { color: #f85149; }

.nc-rtt { color: rgba(255,255,255,0.3); font-size: 10px; }

@keyframes blink { 0%,100% { opacity:1; } 50% { opacity:0.3; } }

/* Body */
.nc-body { flex: 1; overflow: hidden; display: flex; flex-direction: column; }
.nc-stub {
    flex: 1; display: flex; align-items: center; justify-content: center;
    color: rgba(255,255,255,0.2); font-size: 13px; font-style: italic;
}
</style>
