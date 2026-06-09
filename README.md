<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<meta name="theme-color" content="#0f172a"/>
<title>SeedBase RFID — Piloto</title>

<!-- Firebase v10 -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import {
  getFirestore, collection, doc, addDoc, setDoc, getDoc, getDocs,
  query, where, orderBy, limit, updateDoc, serverTimestamp
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

// ══════════════════════════════════════════
// CONFIGURAÇÃO FIREBASE
// Cole aqui os dados do seu projeto Firebase
// Ou configure pela aba ⚙️ Config no app
// ══════════════════════════════════════════
let firebaseConfig = null;
let app = null;
let db = null;

// Cultivares padrão SeedBase 25/26
const CULTIVARES_PADRAO = [
  { id: 'zeus',    nome: '55I57RSF IPRO',  fantasia: 'Zeus',    tech: 'Intacta RR2 Pro' },
  { id: 'grafeno', nome: '57K56RSF CE',    fantasia: 'Grafeno', tech: 'Conkesta E3' },
  { id: 'raca',    nome: '63E66RSF E',     fantasia: 'Raça',    tech: 'Enlist E3' },
  { id: 'c2530',   nome: 'C2530CE',        fantasia: 'C2530',   tech: 'Conkesta E3' },
  { id: 'c2645',   nome: 'C2645CE',        fantasia: 'C2645',   tech: 'Conkesta E3' },
];

const EVENTO_LABEL = {
  vinculacao:          '🏷️ Vinculação de tag',
  entrada_armazem:     '📥 Entrada armazém',
  movimentacao_interna:'↔️ Movimentação interna',
  saida_expedicao:     '🚛 Saída / expedição',
  conferencia_carga:   '🔍 Conferência de carga',
  alerta:              '⚠️ Alerta',
};
const EVENTO_COR = {
  vinculacao:'#3b82f6', entrada_armazem:'#22c55e',
  movimentacao_interna:'#f59e0b', saida_expedicao:'#a855f7',
  conferencia_carga:'#06b6d4', alerta:'#ef4444',
};

// Estado global
let pedidoAtivo = null;
let cargaLidos  = [];
let cargaDivs   = [];
let epcTimer    = null;

// ──────────────────────────────────────────
// INIT
// ──────────────────────────────────────────
window.addEventListener('DOMContentLoaded', () => {
  carregarCamposConfig();
  tentarConectar();
  preencherCultivares();
});

function tentarConectar() {
  const cfg = localStorage.getItem('fb_config');
  if (!cfg) { setConn(false, 'Não configurado'); showBanner(true); return; }
  try {
    firebaseConfig = JSON.parse(cfg);
    app = initializeApp(firebaseConfig, 'seedbase-rfid');
    db  = getFirestore(app);
    testarConexao();
  } catch(e) {
    setConn(false, 'Erro na config');
    showBanner(true);
  }
}

async function testarConexao() {
  try {
    await getDocs(query(collection(db, 'bags'), limit(1)));
    setConn(true, 'Conectado');
    showBanner(false);
  } catch(e) {
    setConn(false, 'Erro Firebase');
    showBanner(true);
  }
}

function setConn(ok, label) {
  document.getElementById('connDot').className = 'conn-dot ' + (ok ? 'ok' : 'err');
  document.getElementById('connLabel').textContent = label;
}
function showBanner(show) {
  document.getElementById('setupBanner').style.display = show ? 'block' : 'none';
}

// ──────────────────────────────────────────
// CONFIG
// ──────────────────────────────────────────
window.salvarConfig = function() {
  const apiKey     = document.getElementById('cfgApiKey').value.trim();
  const projectId  = document.getElementById('cfgProjectId').value.trim();
  const appId      = document.getElementById('cfgAppId').value.trim();
  if (!apiKey || !projectId) { flash('Preencha API Key e Project ID', 'err'); return; }
  const cfg = {
    apiKey,
    authDomain:  projectId + '.firebaseapp.com',
    projectId,
    storageBucket: projectId + '.appspot.com',
    appId: appId || undefined,
  };
  localStorage.setItem('fb_config', JSON.stringify(cfg));
  flash('Salvando...', 'ok');
  setTimeout(() => location.reload(), 800);
};

window.salvarOperador = function() {
  localStorage.setItem('operador',   document.getElementById('cfgOperador').value.trim());
  localStorage.setItem('dispositivo',document.getElementById('cfgDispositivo').value.trim());
  flash('Operador salvo!', 'ok');
};

window.limparConfig = function() {
  if (!confirm('Limpar toda a configuração local?')) return;
  localStorage.clear();
  location.reload();
};

function carregarCamposConfig() {
  const cfg = localStorage.getItem('fb_config');
  if (cfg) {
    const c = JSON.parse(cfg);
    document.getElementById('cfgApiKey').value    = c.apiKey    || '';
    document.getElementById('cfgProjectId').value = c.projectId || '';
    document.getElementById('cfgAppId').value     = c.appId     || '';
  }
  document.getElementById('cfgOperador').value    = localStorage.getItem('operador')    || '';
  document.getElementById('cfgDispositivo').value = localStorage.getItem('dispositivo') || '';
}

function preencherCultivares() {
  const saved = localStorage.getItem('cultivares_extra');
  const extras = saved ? JSON.parse(saved) : [];
  const todos = [...CULTIVARES_PADRAO, ...extras];
  const sel = document.getElementById('cultivarSelect');
  sel.innerHTML = '<option value="">Selecione...</option>' +
    todos.map(c => `<option value="${c.id}">${c.nome}${c.fantasia?' — '+c.fantasia:''}</option>`).join('');
}

function getCultivarNome(id) {
  const saved = localStorage.getItem('cultivares_extra');
  const extras = saved ? JSON.parse(saved) : [];
  const todos = [...CULTIVARES_PADRAO, ...extras];
  const c = todos.find(x => x.id === id);
  return c ? c.nome + (c.fantasia ? ' — ' + c.fantasia : '') : id;
}

// ──────────────────────────────────────────
// NAVEGAÇÃO
// ──────────────────────────────────────────
window.goTab = function(name) {
  document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.getElementById('screen-' + name).classList.add('active');
  const tabs = ['vincular','conferir','buscar','dashboard','config'];
  document.querySelectorAll('.tab')[tabs.indexOf(name)].classList.add('active');
  if (name === 'dashboard') carregarDashboard();
};

// ──────────────────────────────────────────
// LEITURA EPC (modo HID teclado)
// ──────────────────────────────────────────
window.onEpcInput = function(val) {
  clearTimeout(epcTimer);
  const st = document.getElementById('epcStatus');
  if (!val) { st.textContent = ''; return; }
  st.textContent = '⏳ Aguardando leitura completa...';
  st.style.color = 'var(--text3)';
  epcTimer = setTimeout(() => {
    const epc = val.trim().replace(/[\n\r]/g, '');
    if (epc.length >= 4) {
      document.getElementById('epcInput').value = epc;
      st.textContent = '✅ EPC: ' + epc;
      st.style.color = 'var(--green)';
      document.getElementById('numeroBag').focus();
    }
  }, 400);
};

window.onEpcCarga = function(val) {
  clearTimeout(epcTimer);
  epcTimer = setTimeout(async () => {
    const epc = val.trim().replace(/[\n\r]/g, '');
    if (epc.length >= 4 && pedidoAtivo) {
      document.getElementById('epcCarga').value = '';
      await registrarBagNaCarga(epc);
    }
  }, 400);
};

window.clearEpc = function(fieldId) {
  document.getElementById(fieldId || 'epcInput').value = '';
  if (fieldId === 'epcInput' || !fieldId) {
    document.getElementById('epcStatus').textContent = '';
  }
  document.getElementById(fieldId || 'epcInput').focus();
};

// ──────────────────────────────────────────
// VINCULAR BAG
// ──────────────────────────────────────────
window.vincularBag = async function() {
  if (!db) { flash('Configure o Firebase primeiro (aba ⚙️)', 'err'); return; }

  const epc       = document.getElementById('epcInput').value.trim();
  const numero    = document.getElementById('numeroBag').value.trim().toUpperCase();
  const cultivarId= document.getElementById('cultivarSelect').value;

  if (!epc)       { flash('Leia a tag RFID primeiro', 'err'); return; }
  if (!numero)    { flash('Informe o número do bag', 'err'); return; }
  if (!cultivarId){ flash('Selecione a cultivar', 'err'); return; }

  const operador   = localStorage.getItem('operador')    || 'Operador';
  const dispositivo= localStorage.getItem('dispositivo') || 'App';

  // Verificar EPC duplicado
  const snap = await getDocs(query(collection(db,'bags'), where('epc','==',epc)));
  if (!snap.empty) {
    const b = snap.docs[0].data();
    flash(`Tag já vinculada ao bag ${b.numero_bag}`, 'warn');
    return;
  }

  const loc = document.getElementById('localizacao').value.trim() || 'Armazém SCV';

  const bagData = {
    epc,
    numero_bag:   numero,
    cultivar_id:  cultivarId,
    cultivar_nome: getCultivarNome(cultivarId),
    categoria:    document.getElementById('categoriaSelect').value,
    peneira:      document.getElementById('peneira').value.trim(),
    tratamento:   document.getElementById('tratamento').value.trim(),
    peso_kg:      parseFloat(document.getElementById('peso').value) || null,
    produtor:     document.getElementById('produtor').value.trim(),
    localizacao_atual: loc,
    status:       'em_estoque',
    criado_em:    serverTimestamp(),
    atualizado_em: serverTimestamp(),
  };

  // Salvar bag
  const bagRef = await addDoc(collection(db, 'bags'), bagData);

  // Registrar movimentação
  await addDoc(collection(db, 'movimentacoes'), {
    bag_id:     bagRef.id,
    numero_bag: numero,
    tipo_evento:'vinculacao',
    local_destino: loc,
    observacao: `Bag ${numero} vinculado — EPC ${epc}`,
    operador,
    dispositivo,
    criado_em:  serverTimestamp(),
  });

  flash(`✅ Bag ${numero} vinculado!`, 'ok');
  limparFormVincular();
};

function limparFormVincular() {
  ['epcInput','numeroBag','peneira','tratamento','peso','localizacao','produtor'].forEach(id => {
    document.getElementById(id).value = '';
  });
  document.getElementById('epcStatus').textContent = '';
  document.getElementById('cultivarSelect').value = '';
  document.getElementById('categoriaSelect').value = 'C1';
  document.getElementById('epcInput').focus();
}

// ──────────────────────────────────────────
// CONFERÊNCIA DE CARGA
// ──────────────────────────────────────────
window.criarPedido = async function() {
  if (!db) { flash('Configure o Firebase primeiro', 'err'); return; }
  const num = document.getElementById('numPedido').value.trim();
  if (!num) { flash('Informe o número do pedido', 'err'); return; }

  const ref = await addDoc(collection(db, 'pedidos_carga'), {
    numero_pedido:  num,
    placa_veiculo:  document.getElementById('placa').value.trim(),
    motorista:      document.getElementById('motorista').value.trim(),
    status:         'conferindo',
    criado_em:      serverTimestamp(),
  });

  pedidoAtivo = { id: ref.id, numero: num };
  cargaLidos  = [];
  cargaDivs   = [];
  atualizarKpisCarga();

  document.getElementById('conferenciaPainel').style.display = 'block';
  document.getElementById('epcCarga').focus();
  flash(`Pedido ${num} iniciado!`, 'ok');
};

async function registrarBagNaCarga(epc) {
  // Buscar bag pelo EPC
  const snap = await getDocs(query(collection(db,'bags'), where('epc','==',epc)));
  const operador    = localStorage.getItem('operador')    || 'Operador';
  const dispositivo = localStorage.getItem('dispositivo') || 'App';

  if (snap.empty) {
    if (!cargaDivs.find(d => d.epc === epc)) {
      cargaDivs.push({ epc, motivo: 'Tag não cadastrada' });
    }
    atualizarKpisCarga();
    renderDivergencias();
    flash(`⚠️ Tag ${epc} não cadastrada!`, 'warn');
    return;
  }

  const bagDoc  = snap.docs[0];
  const bagData = bagDoc.data();

  if (cargaLidos.find(b => b.epc === epc)) {
    flash('Tag já lida nesta carga', 'warn');
    return;
  }

  // Registrar movimentação
  await addDoc(collection(db,'movimentacoes'), {
    bag_id:      bagDoc.id,
    numero_bag:  bagData.numero_bag,
    tipo_evento: 'conferencia_carga',
    observacao:  `Lido na carga — pedido ${pedidoAtivo.numero}`,
    operador,
    dispositivo,
    criado_em:   serverTimestamp(),
  });

  // Adicionar ao pedido
  await addDoc(collection(db, `pedidos_carga/${pedidoAtivo.id}/itens`), {
    bag_id:           bagDoc.id,
    numero_bag:       bagData.numero_bag,
    epc,
    status_validacao: 'lido',
    criado_em:        serverTimestamp(),
  });

  cargaLidos.push({
    epc,
    numero:   bagData.numero_bag,
    cultivar: bagData.cultivar_nome || '—',
  });

  atualizarKpisCarga();
  renderLidos();
  flash(`✅ ${bagData.numero_bag} — ${bagData.cultivar_nome || ''}`, 'ok');
}

function atualizarKpisCarga() {
  document.getElementById('carga-lidos').textContent = cargaLidos.length;
  document.getElementById('carga-ok').textContent    = cargaLidos.length - cargaDivs.length;
  document.getElementById('carga-div').textContent   = cargaDivs.length;
}

function renderLidos() {
  const el = document.getElementById('carga-lidos-lista');
  if (!cargaLidos.length) { el.innerHTML = '<p style="color:var(--text2);font-size:13px">Nenhum bag lido ainda.</p>'; return; }
  el.innerHTML = cargaLidos.map(b => `
    <div style="display:flex;justify-content:space-between;align-items:center;padding:7px 0;border-bottom:1px solid var(--border);font-size:13px">
      <span style="font-weight:600">${b.numero}</span>
      <span style="color:var(--text2)">${b.cultivar}</span>
      <span class="badge badge-green">OK</span>
    </div>`).join('');
}

function renderDivergencias() {
  const wrap = document.getElementById('carga-lista-div');
  if (!cargaDivs.length) { wrap.style.display='none'; return; }
  wrap.style.display = 'block';
  document.getElementById('carga-divs').innerHTML = cargaDivs.map(d => `
    <div style="display:flex;justify-content:space-between;padding:7px 0;border-bottom:1px solid var(--border);font-size:13px">
      <span style="font-family:monospace;color:var(--red)">${d.epc}</span>
      <span style="color:var(--text2)">${d.motivo}</span>
    </div>`).join('');
}

window.confirmarExpedicao = async function() {
  if (!pedidoAtivo) return;
  if (!cargaLidos.length) { flash('Nenhum bag lido', 'err'); return; }
  if (!confirm(`Confirmar expedição de ${cargaLidos.length} bag(s)?`)) return;

  const operador    = localStorage.getItem('operador')    || 'Operador';
  const dispositivo = localStorage.getItem('dispositivo') || 'App';

  for (const b of cargaLidos) {
    const snap = await getDocs(query(collection(db,'bags'), where('epc','==',b.epc)));
    if (snap.empty) continue;
    const bagRef = snap.docs[0].ref;
    await updateDoc(bagRef, { status:'expedido', atualizado_em: serverTimestamp() });
    await addDoc(collection(db,'movimentacoes'), {
      bag_id:      snap.docs[0].id,
      numero_bag:  b.numero,
      tipo_evento: 'saida_expedicao',
      observacao:  `Expedido — pedido ${pedidoAtivo.numero}`,
      operador,
      dispositivo,
      criado_em:   serverTimestamp(),
    });
  }

  await updateDoc(doc(db,'pedidos_carga', pedidoAtivo.id), {
    status: cargaDivs.length > 0 ? 'divergencia' : 'expedido',
    atualizado_em: serverTimestamp(),
  });

  flash(`✅ ${cargaLidos.length} bags expedidos!`, 'ok');
  window.cancelarPedido();
};

window.cancelarPedido = function() {
  pedidoAtivo = null;
  cargaLidos  = [];
  cargaDivs   = [];
  document.getElementById('conferenciaPainel').style.display = 'none';
  ['numPedido','placa','motorista'].forEach(id => document.getElementById(id).value='');
  atualizarKpisCarga();
};

// ──────────────────────────────────────────
// BUSCAR BAG
// ──────────────────────────────────────────
window.buscarBag = async function() {
  if (!db) { flash('Configure o Firebase primeiro', 'err'); return; }
  const q = document.getElementById('buscaEpc').value.trim().toUpperCase();
  if (!q) { flash('Digite ou leia um EPC / número', 'err'); return; }

  const el = document.getElementById('buscaResultado');
  el.innerHTML = '<p style="color:var(--text2);font-size:13px;padding:12px 0">🔍 Buscando...</p>';

  // Busca por EPC
  let snap = await getDocs(query(collection(db,'bags'), where('epc','==',q)));
  // Se não achou, busca por número do bag
  if (snap.empty) {
    snap = await getDocs(query(collection(db,'bags'), where('numero_bag','==',q)));
  }
  if (snap.empty) {
    el.innerHTML = '<p style="color:var(--red);font-size:13px;padding:12px 0">❌ Bag não encontrado.</p>';
    return;
  }

  const bagDoc  = snap.docs[0];
  const bag     = bagDoc.data();

  // Histórico
  const movSnap = await getDocs(
    query(collection(db,'movimentacoes'),
      where('bag_id','==', bagDoc.id),
      orderBy('criado_em','desc'),
      limit(10))
  );

  const statusBadge = {
    em_estoque: '<span class="badge badge-green">Em estoque</span>',
    expedido:   '<span class="badge badge-amber">Expedido</span>',
    perdido:    '<span class="badge badge-red">Perdido</span>',
  }[bag.status] || `<span class="badge badge-gray">${bag.status}</span>`;

  el.innerHTML = `
    <div class="result-card">
      <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:10px">
        <span style="font-size:17px;font-weight:700">${bag.numero_bag}</span>
        ${statusBadge}
      </div>
      ${row('EPC', `<span style="font-family:monospace;color:var(--blue)">${bag.epc}</span>`)}
      ${row('Cultivar', bag.cultivar_nome || '—')}
      ${row('Categoria', bag.categoria || '—')}
      ${row('Peneira', bag.peneira || '—')}
      ${row('Tratamento', bag.tratamento || '—')}
      ${row('Peso', bag.peso_kg ? bag.peso_kg+' kg' : '—')}
      ${row('Produtor', bag.produtor || '—')}
      ${row('Localização', `<strong>${bag.localizacao_atual || '—'}</strong>`)}
    </div>
    <div class="card" style="margin-top:8px">
      <div class="card-title">📋 Histórico</div>
      ${movSnap.empty ? '<p style="color:var(--text3);font-size:13px">Sem movimentações.</p>' :
        movSnap.docs.map(m => {
          const mv = m.data();
          return `<div class="mov-item">
            <div class="mov-dot" style="background:${EVENTO_COR[mv.tipo_evento]||'#64748b'}"></div>
            <div>
              <div class="mov-event">${EVENTO_LABEL[mv.tipo_evento]||mv.tipo_evento}</div>
              <div class="mov-meta">${fmtData(mv.criado_em)} · ${mv.operador||'—'} · ${mv.dispositivo||''}</div>
              ${mv.observacao?`<div style="font-size:11px;color:var(--text3);margin-top:2px">${mv.observacao}</div>`:''}
            </div>
          </div>`;
        }).join('')
      }
    </div>`;
};

function row(k, v) {
  return `<div class="result-row"><span class="result-key">${k}</span><span class="result-val">${v}</span></div>`;
}

// ──────────────────────────────────────────
// DASHBOARD
// ──────────────────────────────────────────
window.carregarDashboard = async function() {
  if (!db) return;

  // Todos os bags
  const bagsSnap = await getDocs(collection(db,'bags'));
  const bags = bagsSnap.docs.map(d => ({ id:d.id, ...d.data() }));

  const total    = bags.length;
  const estoque  = bags.filter(b => b.status==='em_estoque').length;
  const expedido = bags.filter(b => b.status==='expedido').length;

  document.getElementById('kpi-total').textContent    = total;
  document.getElementById('kpi-estoque').textContent  = estoque;
  document.getElementById('kpi-expedido').textContent = expedido;

  // Últimas movimentações
  const movSnap = await getDocs(
    query(collection(db,'movimentacoes'), orderBy('criado_em','desc'), limit(20))
  );
  const movEl = document.getElementById('movList');
  if (movSnap.empty) {
    movEl.innerHTML = '<p style="color:var(--text2);font-size:13px">Nenhuma movimentação ainda.</p>';
  } else {
    movEl.innerHTML = movSnap.docs.map(d => {
      const m = d.data();
      return `<div class="mov-item">
        <div class="mov-dot" style="background:${EVENTO_COR[m.tipo_evento]||'#64748b'}"></div>
        <div>
          <div class="mov-event">${m.numero_bag||'—'} — ${EVENTO_LABEL[m.tipo_evento]||m.tipo_evento}</div>
          <div class="mov-meta">${fmtData(m.criado_em)} · ${m.operador||'—'}</div>
        </div>
      </div>`;
    }).join('');
  }

  // Lista de bags
  const bagsEl = document.getElementById('bagsList');
  if (!bags.length) {
    bagsEl.innerHTML = '<p style="color:var(--text2);font-size:13px">Nenhum bag cadastrado ainda.</p>';
  } else {
    const sc = { em_estoque:'badge-green', expedido:'badge-amber', perdido:'badge-red' };
    bagsEl.innerHTML = [...bags].reverse().map(b => `
      <div style="display:flex;justify-content:space-between;align-items:center;padding:7px 0;border-bottom:1px solid var(--border);font-size:13px;gap:6px;flex-wrap:wrap">
        <span style="font-weight:600;min-width:75px">${b.numero_bag}</span>
        <span style="color:var(--text2);flex:1">${b.cultivar_nome||'—'}</span>
        <span style="color:var(--text3);font-size:11px">${b.localizacao_atual||''}</span>
        <span class="badge ${sc[b.status]||'badge-gray'}">${(b.status||'').replace('_',' ')}</span>
      </div>`).join('');
  }
};

// ──────────────────────────────────────────
// UTILITÁRIOS
// ──────────────────────────────────────────
function fmtData(ts) {
  if (!ts) return '—';
  const d = ts.toDate ? ts.toDate() : new Date(ts);
  return d.toLocaleDateString('pt-BR') + ' ' +
         d.toLocaleTimeString('pt-BR', { hour:'2-digit', minute:'2-digit' });
}

let flashTimer = null;
window.flash = function(msg, tipo) {
  const el = document.getElementById('flash');
  el.textContent = msg;
  el.className = `flash ${tipo} show`;
  clearTimeout(flashTimer);
  flashTimer = setTimeout(() => el.classList.remove('show'), 3200);
};
</script>

<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
:root{
  --bg:#0f172a;--bg2:#1e293b;--bg3:#334155;
  --border:#334155;--border2:#475569;
  --text:#f1f5f9;--text2:#94a3b8;--text3:#64748b;
  --green:#22c55e;--green-bg:rgba(34,197,94,.12);
  --amber:#f59e0b;--amber-bg:rgba(245,158,11,.12);
  --red:#ef4444;--red-bg:rgba(239,68,68,.12);
  --blue:#3b82f6;--blue-bg:rgba(59,130,246,.12);
  --radius:10px;--radius-sm:6px;
}
body{background:var(--bg);color:var(--text);font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif;font-size:15px;min-height:100vh}

.header{background:var(--bg2);border-bottom:1px solid var(--border);padding:12px 16px;display:flex;align-items:center;justify-content:space-between;position:sticky;top:0;z-index:100}
.logo{font-size:17px;font-weight:700;letter-spacing:-.3px}
.logo span{color:var(--blue)}
.conn-dot{width:8px;height:8px;border-radius:50%;background:var(--text3);display:inline-block;margin-right:6px;transition:background .3s}
.conn-dot.ok{background:var(--green)}
.conn-dot.err{background:var(--red)}
.conn-label{font-size:12px;color:var(--text2)}

.tabs{display:flex;background:var(--bg2);border-bottom:1px solid var(--border);overflow-x:auto;-webkit-overflow-scrolling:touch}
.tab{flex:1;min-width:64px;padding:11px 4px;text-align:center;font-size:11px;font-weight:500;color:var(--text3);cursor:pointer;border-bottom:2px solid transparent;transition:all .15s;white-space:nowrap}
.tab.active{color:var(--blue);border-bottom-color:var(--blue)}
.tab-icon{display:block;font-size:17px;margin-bottom:2px}

.screen{display:none;padding:16px;max-width:600px;margin:0 auto}
.screen.active{display:block}

.card{background:var(--bg2);border:1px solid var(--border);border-radius:var(--radius);padding:14px;margin-bottom:12px}
.card-title{font-size:12px;font-weight:600;color:var(--text2);margin-bottom:12px;text-transform:uppercase;letter-spacing:.5px}

label{display:block;font-size:12px;color:var(--text2);margin-bottom:5px;font-weight:500}
input,select{width:100%;background:var(--bg3);border:1px solid var(--border2);border-radius:var(--radius-sm);padding:10px 12px;color:var(--text);font-size:15px;outline:none;transition:border .15s}
input:focus,select:focus{border-color:var(--blue)}
select option{background:var(--bg2)}
.form-group{margin-bottom:12px}
.form-row{display:grid;grid-template-columns:1fr 1fr;gap:10px}

.epc-wrap{position:relative}
.epc-field{font-family:monospace;font-size:16px;font-weight:700;letter-spacing:1px;padding-right:44px;background:var(--bg);border:2px solid var(--blue);color:var(--blue)}
.epc-field::placeholder{color:var(--text3);font-weight:400;font-size:13px;letter-spacing:0}
.epc-clear{position:absolute;right:10px;top:50%;transform:translateY(-50%);background:none;border:none;color:var(--text3);cursor:pointer;font-size:18px;padding:4px;line-height:1}

.btn{display:flex;align-items:center;justify-content:center;gap:6px;padding:12px 18px;border-radius:var(--radius-sm);font-size:14px;font-weight:600;cursor:pointer;border:none;transition:opacity .15s;width:100%}
.btn+.btn{margin-top:8px}
.btn-primary{background:var(--blue);color:#fff}
.btn-success{background:var(--green);color:#fff}
.btn-ghost{background:var(--bg3);color:var(--text);border:1px solid var(--border2)}
.btn:active{opacity:.8}

.badge{display:inline-block;padding:2px 8px;border-radius:4px;font-size:11px;font-weight:600}
.badge-green{background:var(--green-bg);color:var(--green)}
.badge-amber{background:var(--amber-bg);color:var(--amber)}
.badge-red{background:var(--red-bg);color:var(--red)}
.badge-blue{background:var(--blue-bg);color:var(--blue)}
.badge-gray{background:var(--bg3);color:var(--text2)}

.kpi-row{display:grid;grid-template-columns:repeat(3,1fr);gap:8px;margin-bottom:12px}
.kpi{background:var(--bg2);border:1px solid var(--border);border-radius:var(--radius);padding:12px;text-align:center}
.kpi-val{font-size:26px;font-weight:700}
.kpi-lbl{font-size:11px;color:var(--text2);margin-top:3px}

.mov-item{display:flex;gap:12px;padding:10px 0;border-bottom:1px solid var(--border)}
.mov-item:last-child{border-bottom:none}
.mov-dot{width:10px;height:10px;border-radius:50%;flex-shrink:0;margin-top:4px}
.mov-event{font-size:13px;font-weight:500}
.mov-meta{font-size:11px;color:var(--text3);margin-top:2px}

.result-card{background:var(--bg3);border:1px solid var(--border2);border-radius:var(--radius);padding:14px;margin-top:12px}
.result-row{display:flex;justify-content:space-between;padding:5px 0;border-bottom:1px solid var(--border);font-size:13px}
.result-row:last-child{border-bottom:none}
.result-key{color:var(--text2)}
.result-val{font-weight:500;text-align:right;max-width:60%}

.setup-banner{background:var(--amber-bg);border:1px solid var(--amber);border-radius:var(--radius);padding:14px;margin-bottom:16px}
.setup-banner h3{color:var(--amber);font-size:14px;margin-bottom:6px}
.setup-banner p{font-size:12px;color:var(--text2);line-height:1.6}
.setup-banner code{background:var(--bg3);padding:2px 6px;border-radius:4px;font-size:11px}

.flash{position:fixed;bottom:24px;left:50%;transform:translateX(-50%);padding:12px 20px;border-radius:var(--radius);font-size:14px;font-weight:500;z-index:999;opacity:0;transition:opacity .25s;pointer-events:none;min-width:240px;text-align:center;white-space:nowrap}
.flash.show{opacity:1}
.flash.ok{background:var(--green);color:#fff}
.flash.err{background:var(--red);color:#fff}
.flash.warn{background:var(--amber);color:#000}

.epc-status{font-size:12px;color:var(--text3);margin-top:-6px;margin-bottom:8px;min-height:18px}
</style>
</head>
<body>

<div class="flash" id="flash"></div>

<!-- HEADER -->
<div class="header">
  <div class="logo">Seed<span>Base</span> RFID</div>
  <div>
    <span class="conn-dot" id="connDot"></span>
    <span class="conn-label" id="connLabel">Conectando...</span>
  </div>
</div>

<!-- TABS -->
<div class="tabs">
  <div class="tab active" onclick="goTab('vincular')">
    <span class="tab-icon">🏷️</span>Vincular
  </div>
  <div class="tab" onclick="goTab('conferir')">
    <span class="tab-icon">🚛</span>Conferir
  </div>
  <div class="tab" onclick="goTab('buscar')">
    <span class="tab-icon">🔍</span>Buscar
  </div>
  <div class="tab" onclick="goTab('dashboard')">
    <span class="tab-icon">📊</span>Estoque
  </div>
  <div class="tab" onclick="goTab('config')">
    <span class="tab-icon">⚙️</span>Config
  </div>
</div>

<!-- ══ TELA 1 — VINCULAR ══ -->
<div class="screen active" id="screen-vincular">

  <div class="setup-banner" id="setupBanner" style="display:none">
    <h3>⚠️ Configure o Firebase primeiro</h3>
    <p>Vá na aba <strong>⚙️ Config</strong>, cole os dados do seu projeto Firebase e toque em <strong>Salvar e conectar</strong>.</p>
  </div>

  <div class="card">
    <div class="card-title">📡 Tag RFID</div>
    <div class="form-group">
      <label>EPC da tag — aponte o leitor ou digite</label>
      <div class="epc-wrap">
        <input class="epc-field" id="epcInput" type="text"
          placeholder="Aguardando leitor..."
          autocomplete="off" autocorrect="off" spellcheck="false"
          oninput="onEpcInput(this.value)"/>
        <button class="epc-clear" onclick="clearEpc('epcInput')">✕</button>
      </div>
      <div class="epc-status" id="epcStatus"></div>
    </div>
  </div>

  <div class="card">
    <div class="card-title">📦 Dados do bag</div>
    <div class="form-group">
      <label>Número do bag / lote *</label>
      <input id="numeroBag" type="text" placeholder="Ex: 26S0001"/>
    </div>
    <div class="form-group">
      <label>Cultivar *</label>
      <select id="cultivarSelect"></select>
    </div>
    <div class="form-row">
      <div class="form-group">
        <label>Categoria</label>
        <select id="categoriaSelect">
          <option value="C1">C1</option>
          <option value="C2">C2</option>
          <option value="S1">S1</option>
          <option value="S2">S2</option>
        </select>
      </div>
      <div class="form-group">
        <label>Peneira</label>
        <input id="peneira" type="text" placeholder="Ex: Redonda 5,5"/>
      </div>
    </div>
    <div class="form-row">
      <div class="form-group">
        <label>Peso (kg)</label>
        <input id="peso" type="number" placeholder="40" step="0.1"/>
      </div>
      <div class="form-group">
        <label>Tratamento</label>
        <input id="tratamento" type="text" placeholder="Ex: Standak Top"/>
      </div>
    </div>
    <div class="form-group">
      <label>Produtor</label>
      <input id="produtor" type="text" placeholder="Nome do produtor"/>
    </div>
    <div class="form-group">
      <label>Localização inicial</label>
      <input id="localizacao" type="text" placeholder="Ex: SCV — Rua A1, Pos 01"/>
    </div>
  </div>

  <button class="btn btn-primary" onclick="vincularBag()">🔗 Vincular tag ao bag</button>
  <button class="btn btn-ghost" onclick="goTab('buscar')">🔍 Buscar bag existente</button>
</div>

<!-- ══ TELA 2 — CONFERIR CARGA ══ -->
<div class="screen" id="screen-conferir">
  <div class="card">
    <div class="card-title">🚛 Novo carregamento</div>
    <div class="form-group">
      <label>Número do pedido / NF</label>
      <input id="numPedido" type="text" placeholder="Ex: NF 001238"/>
    </div>
    <div class="form-row">
      <div class="form-group">
        <label>Placa</label>
        <input id="placa" type="text" placeholder="ABC-1234"/>
      </div>
      <div class="form-group">
        <label>Motorista</label>
        <input id="motorista" type="text" placeholder="Nome"/>
      </div>
    </div>
    <button class="btn btn-primary" onclick="criarPedido()">📋 Iniciar conferência</button>
  </div>

  <div id="conferenciaPainel" style="display:none">
    <div class="card">
      <div class="card-title">📡 Leitura dos bags</div>
      <p style="font-size:13px;color:var(--text2);margin-bottom:10px">Aponte o leitor para cada bag carregado.</p>
      <div class="epc-wrap">
        <input class="epc-field" id="epcCarga" type="text"
          placeholder="Aguardando leitor..."
          autocomplete="off" autocorrect="off" spellcheck="false"
          oninput="onEpcCarga(this.value)"/>
        <button class="epc-clear" onclick="clearEpc('epcCarga')">✕</button>
      </div>
    </div>

    <div class="kpi-row">
      <div class="kpi">
        <div class="kpi-val" id="carga-lidos" style="color:var(--green)">0</div>
        <div class="kpi-lbl">Lidos</div>
      </div>
      <div class="kpi">
        <div class="kpi-val" id="carga-ok" style="color:var(--blue)">0</div>
        <div class="kpi-lbl">OK</div>
      </div>
      <div class="kpi">
        <div class="kpi-val" id="carga-div" style="color:var(--red)">0</div>
        <div class="kpi-lbl">Divergências</div>
      </div>
    </div>

    <div class="card" id="carga-lista-div" style="display:none">
      <div class="card-title" style="color:var(--red)">⚠️ Divergências</div>
      <div id="carga-divs"></div>
    </div>

    <div class="card">
      <div class="card-title">✅ Bags lidos</div>
      <div id="carga-lidos-lista"><p style="color:var(--text2);font-size:13px">Nenhum bag lido ainda.</p></div>
    </div>

    <button class="btn btn-success" onclick="confirmarExpedicao()">✅ Confirmar expedição</button>
    <button class="btn btn-ghost" onclick="cancelarPedido()">✕ Cancelar</button>
  </div>
</div>

<!-- ══ TELA 3 — BUSCAR ══ -->
<div class="screen" id="screen-buscar">
  <div class="card">
    <div class="card-title">🔍 Buscar bag</div>
    <div class="epc-wrap" style="margin-bottom:10px">
      <input class="epc-field" id="buscaEpc" type="text"
        placeholder="EPC ou número do bag..."
        autocomplete="off" autocorrect="off" spellcheck="false"/>
      <button class="epc-clear" onclick="clearEpc('buscaEpc');document.getElementById('buscaResultado').innerHTML=''">✕</button>
    </div>
    <button class="btn btn-primary" onclick="buscarBag()">🔍 Buscar</button>
  </div>
  <div id="buscaResultado"></div>
</div>

<!-- ══ TELA 4 — DASHBOARD ══ -->
<div class="screen" id="screen-dashboard">
  <div class="kpi-row">
    <div class="kpi">
      <div class="kpi-val" id="kpi-total" style="color:var(--blue)">—</div>
      <div class="kpi-lbl">Total bags</div>
    </div>
    <div class="kpi">
      <div class="kpi-val" id="kpi-estoque" style="color:var(--green)">—</div>
      <div class="kpi-lbl">Em estoque</div>
    </div>
    <div class="kpi">
      <div class="kpi-val" id="kpi-expedido" style="color:var(--amber)">—</div>
      <div class="kpi-lbl">Expedidos</div>
    </div>
  </div>
  <div class="card">
    <div class="card-title">🕐 Últimas movimentações</div>
    <div id="movList" style="color:var(--text2);font-size:13px">Carregando...</div>
  </div>
  <div class="card">
    <div class="card-title">📦 Bags cadastrados</div>
    <div id="bagsList" style="color:var(--text2);font-size:13px">Carregando...</div>
  </div>
  <button class="btn btn-ghost" onclick="carregarDashboard()">🔄 Atualizar</button>
</div>

<!-- ══ TELA 5 — CONFIG ══ -->
<div class="screen" id="screen-config">
  <div class="card">
    <div class="card-title">🔥 Firebase</div>
    <p style="font-size:12px;color:var(--text2);margin-bottom:12px;line-height:1.6">
      Acesse <strong>console.firebase.google.com</strong> → seu projeto → <strong>⚙️ Configurações do projeto → Seus apps → SDK</strong> e copie os valores abaixo.
    </p>
    <div class="form-group">
      <label>API Key</label>
      <input id="cfgApiKey" type="text" placeholder="AIzaSy..."/>
    </div>
    <div class="form-group">
      <label>Project ID</label>
      <input id="cfgProjectId" type="text" placeholder="meu-projeto-12345"/>
    </div>
    <div class="form-group">
      <label>App ID (opcional)</label>
      <input id="cfgAppId" type="text" placeholder="1:123456:web:abc..."/>
    </div>
    <button class="btn btn-primary" onclick="salvarConfig()">💾 Salvar e conectar</button>
  </div>

  <div class="card">
    <div class="card-title">👤 Operador</div>
    <div class="form-group">
      <label>Seu nome</label>
      <input id="cfgOperador" type="text" placeholder="Ex: Ruan"/>
    </div>
    <div class="form-group">
      <label>Dispositivo</label>
      <input id="cfgDispositivo" type="text" placeholder="Ex: Android SCV"/>
    </div>
    <button class="btn btn-ghost" onclick="salvarOperador()">💾 Salvar</button>
  </div>

  <div class="card">
    <div class="card-title">📡 Leitor RFID — modo HID teclado</div>
    <p style="font-size:13px;color:var(--text2);line-height:1.7">
      1. Configure o leitor BLE no modo <strong>HID Keyboard</strong><br>
      2. Pareie via Bluetooth nas configurações do celular<br>
      3. Abra a aba <strong>Vincular</strong> e toque no campo EPC<br>
      4. Aponte o leitor para o bag — o código aparece sozinho ✅
    </p>
    <div style="margin-top:10px;padding:10px;background:var(--bg3);border-radius:var(--radius-sm)">
      <div style="font-size:12px;color:var(--amber);font-weight:600;margin-bottom:3px">⚠️ Dica</div>
      <div style="font-size:12px;color:var(--text2)">Sempre toque no campo EPC antes de usar o leitor para garantir o foco correto.</div>
    </div>
  </div>

  <div class="card" style="border-color:var(--red)">
    <div class="card-title" style="color:var(--red)">🗑️ Resetar</div>
    <button class="btn btn-ghost" onclick="limparConfig()">Limpar configuração local</button>
  </div>
</div>

</body>
</html>
