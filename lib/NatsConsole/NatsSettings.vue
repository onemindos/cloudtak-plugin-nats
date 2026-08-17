<template>
    <div class='nats-settings'>
        <div class='ns-header'>
            <Settings :size='14' class='ns-icon' />
            <span class='ns-title'>NATS Connections</span>
        </div>

        <!-- Profile list -->
        <div class='ns-profiles'>
            <div
                v-for='p in profiles'
                :key='p.id'
                class='ns-profile'
                :class='{ active: p.id === activeProfileId, connected: status === 'connected' && p.id === activeProfileId }'
            >
                <div class='nsp-row'>
                    <div class='nsp-info'>
                        <span class='nsp-label'>{{ p.label }}</span>
                        <span class='nsp-url'>{{ p.wsUrl }}</span>
                    </div>
                    <div class='nsp-actions'>
                        <button
                            v-if='p.id !== activeProfileId || status !== 'connected''
                            class='nsp-btn nsp-btn-connect'
                            :disabled='status === 'connecting''
                            @click='connectTo(p.id)'
                        >Connect</button>
                        <button
                            v-else
                            class='nsp-btn nsp-btn-disconnect'
                            @click='disconnect()'
                        >Disconnect</button>
                        <button class='nsp-btn nsp-btn-edit' @click='startEdit(p)'><Pencil :size='11' /></button>
                        <button class='nsp-btn nsp-btn-del' :disabled='profiles.length <= 1' @click='removeProfile(p.id)'><Trash2 :size='11' /></button>
                    </div>
                </div>
            </div>
        </div>

        <!-- Add / Edit form -->
        <div class='ns-form'>
            <div class='ns-form-title'>{{ editingId ? 'Edit Profile' : 'Add Profile' }}</div>
            <div class='ns-fields'>
                <input v-model='form.label' class='ns-input' placeholder='Label (e.g. OneMind Hub)' />
                <input v-model='form.wsUrl' class='ns-input' placeholder='WebSocket URL (e.g. wss://ws.onemindos.dev)' />
                <input v-model='form.user'  class='ns-input' placeholder='User (optional)' />
                <input v-model='form.pass'  class='ns-input' type='password' placeholder='Password (optional)' />
            </div>
            <div class='ns-form-actions'>
                <button class='ns-btn ns-btn-save' :disabled='!form.label || !form.wsUrl' @click='saveForm'>
                    {{ editingId ? 'Update' : 'Add' }}
                </button>
                <button v-if='editingId' class='ns-btn' @click='cancelEdit'>Cancel</button>
            </div>
        </div>

        <!-- RTT / error display -->
        <div v-if='status === 'error' && error' class='ns-error'>
            <AlertCircle :size='13' /> {{ error }}
        </div>
        <div v-if='status === 'connected' && rtt' class='ns-rtt'>
            RTT: {{ rtt }}ms
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue';
import { Settings, Pencil, Trash2, AlertCircle } from 'lucide-vue-next';
import { useNatsStore } from '../stores/nats.store';
import type { NatsProfile } from '../stores/nats.store';

const { status, error, rtt, profiles, activeProfileId, connect, disconnect, addProfile, updateProfile, removeProfile } = useNatsStore();

// Form state
const editingId = ref<string | null>(null);
const form = reactive({ label: '', wsUrl: '', user: '', pass: '' });

function startEdit(p: NatsProfile) {
    editingId.value = p.id;
    form.label = p.label;
    form.wsUrl = p.wsUrl;
    form.user  = p.user  ?? '';
    form.pass  = p.pass  ?? '';
}

function cancelEdit() {
    editingId.value = null;
    Object.assign(form, { label: '', wsUrl: '', user: '', pass: '' });
}

function saveForm() {
    if (editingId.value) {
        updateProfile({
            id:    editingId.value,
            label: form.label,
            wsUrl: form.wsUrl,
            user:  form.user  || undefined,
            pass:  form.pass  || undefined,
        });
    } else {
        addProfile({
            id:    crypto.randomUUID(),
            label: form.label,
            wsUrl: form.wsUrl,
            user:  form.user  || undefined,
            pass:  form.pass  || undefined,
        });
    }
    cancelEdit();
}

function connectTo(id: string) { connect(id); }
</script>

<style scoped>
.nats-settings {
    display: flex; flex-direction: column; height: 100%; overflow-y: auto;
    background: var(--cloudtak-panel-bg, rgba(13,17,23,0.98));
    color: #e6edf3; font-size: 12px; padding: 12px;
    gap: 14px;
}
.ns-header { display: flex; align-items: center; gap: 7px; }
.ns-icon { color: #4a9eff; }
.ns-title { font-weight: 700; font-size: 13px; }

.ns-profiles { display: flex; flex-direction: column; gap: 6px; }
.ns-profile {
    background: rgba(255,255,255,0.04);
    border: 1px solid rgba(255,255,255,0.08);
    border-radius: 7px; padding: 8px 10px;
    transition: border-color .12s;
}
.ns-profile.connected { border-color: rgba(34,197,94,0.35); }

.nsp-row { display: flex; align-items: center; justify-content: space-between; gap: 8px; }
.nsp-info { display: flex; flex-direction: column; gap: 2px; }
.nsp-label { font-weight: 600; font-size: 12px; }
.nsp-url   { font-family: monospace; font-size: 10px; color: rgba(255,255,255,0.35); }
.nsp-actions { display: flex; gap: 4px; }

.nsp-btn {
    background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.1);
    border-radius: 5px; color: rgba(255,255,255,0.5); padding: 3px 8px;
    font-size: 10px; cursor: pointer; display: flex; align-items: center; gap: 3px;
    transition: background .12s, color .12s;
}
.nsp-btn:hover { background: rgba(255,255,255,0.1); color: #e6edf3; }
.nsp-btn:disabled { opacity: 0.3; cursor: default; }
.nsp-btn-connect    { color: #22c55e; border-color: rgba(34,197,94,0.3); }
.nsp-btn-disconnect { color: #f59e0b; border-color: rgba(245,158,11,0.3); }
.nsp-btn-del:hover  { color: #f85149; }

/* Add form */
.ns-form {
    background: rgba(255,255,255,0.03);
    border: 1px solid rgba(255,255,255,0.07);
    border-radius: 8px; padding: 10px 12px;
    display: flex; flex-direction: column; gap: 8px;
}
.ns-form-title { font-size: 11px; font-weight: 600; color: rgba(255,255,255,0.5); }
.ns-fields { display: flex; flex-direction: column; gap: 5px; }
.ns-input {
    background: rgba(255,255,255,0.06);
    border: 1px solid rgba(255,255,255,0.1);
    border-radius: 5px; color: #e6edf3; padding: 5px 9px;
    font-size: 12px; outline: none; width: 100%;
}
.ns-input:focus { border-color: #4a9eff; }
.ns-input::placeholder { color: rgba(255,255,255,0.2); }

.ns-form-actions { display: flex; gap: 6px; }
.ns-btn {
    background: rgba(255,255,255,0.07); border: 1px solid rgba(255,255,255,0.12);
    border-radius: 5px; color: rgba(255,255,255,0.6); padding: 4px 12px;
    font-size: 11px; cursor: pointer;
}
.ns-btn:hover { background: rgba(255,255,255,0.12); color: #e6edf3; }
.ns-btn:disabled { opacity: 0.3; cursor: default; }
.ns-btn-save { color: #4a9eff; border-color: rgba(74,158,255,0.3); background: rgba(74,158,255,0.1); }

.ns-error { color: #f85149; font-size: 11px; display: flex; align-items: center; gap: 5px; }
.ns-rtt   { color: rgba(255,255,255,0.3); font-size: 11px; }
</style>
