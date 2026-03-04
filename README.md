<!doctype html>
<html lang="pt-BR" class="h-full">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Cardápio Online</title>

  <script src="https://cdn.tailwindcss.com/3.4.17"></script>

  <!-- Se você usa Element SDK no seu editor/painel, mantenha estes dois -->
  <script src="/_sdk/element_sdk.js"></script>
  <script src="/_sdk/data_sdk.js" type="text/javascript"></script>

  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@600;700&display=swap" rel="stylesheet" />

  <style>
    * { box-sizing: border-box; }
    body { font-family: 'Poppins', sans-serif; }
    .menu-card { transition: transform .2s, box-shadow .2s; cursor: pointer; }
    .menu-card:hover { transform: translateY(-4px); box-shadow: 0 8px 20px rgba(0,0,0,.1); }
  </style>
</head>

<body class="h-full">
  <div class="w-full h-full overflow-auto" style="background: linear-gradient(135deg, #f5f5f5 0%, #e8e8e8 100%);">
    <!-- Header -->
    <header class="text-center py-6 px-4" style="background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%);">
      <h1 id="restaurant-name" class="text-3xl font-bold text-white">🍽️ Marmitex do Dia</h1>
      <p class="text-white mt-1 text-sm" id="header-subtitle">Peça pelo WhatsApp</p>
      <p class="text-white/90 mt-1 text-xs" id="admin-hint" style="display:none;">
        Modo ADM ativo — URL com <b>#admin</b>
      </p>
    </header>

    <main class="max-w-6xl mx-auto px-4 py-8">
      <!-- Abas da semana -->
      <div class="flex flex-wrap gap-2 justify-center mb-6" id="tabs-semana"></div>

      <!-- Título -->
      <div class="text-center mb-6">
        <h2 id="titulo-dia" class="text-xl font-bold text-gray-800"></h2>
        <p id="subtitulo-dia" class="text-sm text-gray-500 mt-1"></p>
      </div>

      <!-- Conteúdo -->
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6" id="menu-container"></div>
    </main>

    <!-- Footer -->
    <footer class="text-center py-6 px-4 mt-8">
      <p class="text-gray-500 text-xs">© Cardápio Online - Peça agora</p>
    </footer>
  </div>

  <script>
    /*********************
     * CONFIG GERAL
     *********************/
    const defaultConfig = {
      restaurant_name: '🍽️ Marmitex do Dia',
      whatsapp_number: '31991474252', // seu WhatsApp (com DDD)
      min_acompanhamentos: 0,         // 0 = não obriga / 2 = obriga escolher 2
    };
    let currentConfig = { ...defaultConfig };

    // Senha do ADM (troque!)
    const ADMIN_PASSWORD = "1234";

    /*********************
     * DIAS DA SEMANA (SEG-SÁB)
     *********************/
    const diasSemana = [
      { key: 'segunda', label: 'Segunda' },
      { key: 'terca',   label: 'Terça' },
      { key: 'quarta',  label: 'Quarta' },
      { key: 'quinta',  label: 'Quinta' },
      { key: 'sexta',   label: 'Sexta' },
      { key: 'sabado',  label: 'Sábado' },
    ];

    let diaSelecionado = 'segunda';

    /*********************
     * CARDÁPIO (SALVO LOCAL)
     * Cada item do dia:
     * { id, name, desc, sizes: [{size, price}, ...] }
     *********************/
    const MENU_KEY = "menuSemana_v1";

    function defaultMenuSemana() {
      return {
        segunda: [],
        terca: [],
        quarta: [],
        quinta: [],
        sexta: [],
        sabado: [],
      };
    }

    function loadMenuSemana() {
      try {
        const raw = localStorage.getItem(MENU_KEY);
        if (raw) return JSON.parse(raw);
      } catch (e) {}
      return defaultMenuSemana();
    }

    function saveMenuSemana(data) {
      localStorage.setItem(MENU_KEY, JSON.stringify(data));
    }

    let menuSemana = loadMenuSemana();

    /*********************
     * ACOMPANHAMENTOS (CLIENTE)
     *********************/
    const acompanhamentos = [
      'Arroz',
      'Feijão',
      'Espaguete ao molho simples',
      'Chuchu com carne moída',
      'Couve refogada'
    ];

    /*********************
     * UTILITÁRIOS
     *********************/
    function escapeHtml(str) {
      return String(str ?? '')
        .replaceAll('&', '&amp;')
        .replaceAll('<', '&lt;')
        .replaceAll('>', '&gt;')
        .replaceAll('"', '&quot;')
        .replaceAll("'", '&#039;');
    }

    function onlyDigits(str) {
      return String(str || '').replace(/\D/g, '');
    }

    function normalizeWhatsappNumber(raw) {
      let n = onlyDigits(raw);
      if (!n) return '';
      if (!n.startsWith('55')) n = '55' + n; // Brasil
      return n;
    }

    function formatBRL(value) {
      return Number(value || 0).toFixed(2).replace('.', ',');
    }

    function toBRL(n) {
      return `R$ ${Number(n || 0).toFixed(2).replace('.', ',')}`;
    }

    async function copyToClipboard(text) {
      try {
        await navigator.clipboard.writeText(text);
        alert('✅ Pedido copiado! Cole no WhatsApp se necessário.');
      } catch (e) {
        const ta = document.createElement('textarea');
        ta.value = text;
        document.body.appendChild(ta);
        ta.select();
        document.execCommand('copy');
        ta.remove();
        alert('✅ Pedido copiado! Cole no WhatsApp se necessário.');
      }
    }

    /*********************
     * SALVAR DADOS DO CLIENTE (pra não digitar toda vez)
     *********************/
    const CUSTOMER_KEY = "customer_v1";
    function loadCustomer() {
      try { return JSON.parse(localStorage.getItem(CUSTOMER_KEY) || "{}"); }
      catch { return {}; }
    }
    function saveCustomer(data) {
      localStorage.setItem(CUSTOMER_KEY, JSON.stringify(data));
    }

    /*********************
     * VENDAS / FATURAMENTO (LOCAL)
     *********************/
    const SALES_KEY = "sales_log_v1";

    function todayKey() {
      const d = new Date();
      const yyyy = d.getFullYear();
      const mm = String(d.getMonth() + 1).padStart(2, "0");
      const dd = String(d.getDate()).padStart(2, "0");
      return `${yyyy}-${mm}-${dd}`;
    }

    function loadSales() {
      try { return JSON.parse(localStorage.getItem(SALES_KEY) || "[]"); }
      catch { return []; }
    }

    function saveSales(list) {
      localStorage.setItem(SALES_KEY, JSON.stringify(list));
    }

    function addSale(entry) {
      const list = loadSales();
      list.push(entry);
      saveSales(list);
    }

    function salesOfToday() {
      const key = todayKey();
      return loadSales().filter(s => s.dayKey === key);
    }

    function totalsOfToday() {
      const sales = salesOfToday();
      const total = sales.reduce((sum, s) => sum + (Number(s.total || 0)), 0);
      return {
        count: sales.length,
        total,
        avg: sales.length ? total / sales.length : 0,
      };
    }

    function exportTodayCSV() {
      const sales = salesOfToday();
      const header = ["DataHora","Cliente","Telefone","Endereco","Prato","Tamanho","Qtd","Total","Obs"].join(",");
      const lines = sales.map(s => {
        const safe = (v) => `"${String(v ?? "").replaceAll('"','""')}"`;
        return [
          safe(new Date(s.ts).toLocaleString("pt-BR")),
          safe(s.clienteNome),
          safe(s.clienteTelefone),
          safe(s.clienteEndereco),
          safe(s.itemName),
          safe(s.size),
          safe(s.qty),
          safe(String(s.total).replace(".", ",")),
          safe(s.observacoes),
        ].join(",");
      });
      const csv = [header, ...lines].join("\n");
      const blob = new Blob([csv], { type: "text/csv;charset=utf-8;" });
      const url = URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `faturamento_${todayKey()}.csv`;
      document.body.appendChild(a);
      a.click();
      a.remove();
      URL.revokeObjectURL(url);
    }

    function clearTodaySales() {
      const key = todayKey();
      const list = loadSales().filter(s => s.dayKey !== key);
      saveSales(list);
    }

    /*********************
     * MODO ADMIN
     *********************/
    function isAdminMode() {
      return location.hash === "#admin";
    }

    function requireAdminLogin() {
      const ok = sessionStorage.getItem("admin_ok") === "1";
      if (ok) return true;

      const pass = prompt("🔐 Área do ADM\nDigite a senha:");
      if (pass === ADMIN_PASSWORD) {
        sessionStorage.setItem("admin_ok", "1");
        return true;
      }
      alert("❌ Senha incorreta!");
      location.hash = "";
      return false;
    }

    function sairAdmin() {
      sessionStorage.removeItem("admin_ok");
      location.hash = "";
      startApp();
    }

    /*********************
     * RENDER TABS
     *********************/
    function renderTabs() {
      const tabs = document.getElementById('tabs-semana');
      tabs.innerHTML = diasSemana.map(d => `
        <button
          type="button"
          onclick="selecionarDia('${d.key}')"
          class="px-4 py-2 rounded-full font-semibold transition"
          style="
            border: 1px solid ${diaSelecionado === d.key ? '#ff6b35' : '#ddd'};
            background: ${diaSelecionado === d.key ? 'linear-gradient(135deg, #ff6b35 0%, #f7931e 100%)' : '#fff'};
            color: ${diaSelecionado === d.key ? '#fff' : '#333'};
          "
          aria-pressed="${diaSelecionado === d.key ? 'true' : 'false'}"
        >
          ${d.label}
        </button>
      `).join('');
    }

    function selecionarDia(key) {
      diaSelecionado = key;
      renderTabs();
      if (isAdminMode()) renderAdmin();
      else renderMenuDoDia();
    }

    /*********************
     * CLIENTE: RENDER DO DIA
     *********************/
    function renderMenuDoDia() {
      const container = document.getElementById('menu-container');
      const titulo = document.getElementById('titulo-dia');
      const subtitulo = document.getElementById('subtitulo-dia');

      const diaObj = diasSemana.find(d => d.key === diaSelecionado);
      titulo.textContent = `Cardápio de ${diaObj ? diaObj.label : 'Hoje'}`;
      subtitulo.textContent = 'Escolha um dia acima para ver o cardápio.';

      const itens = (menuSemana[diaSelecionado] || []).slice();

      if (!itens.length) {
        container.innerHTML = `
          <div class="bg-white rounded-lg p-6 shadow-md text-center col-span-full"
               style="border-top: 4px solid #ff6b35;">
            <div style="font-size: 40px;">🕒</div>
            <h3 class="text-lg font-bold text-gray-800 mt-2">Em breve!</h3>
            <p class="text-sm text-gray-500 mt-1">
              O cardápio deste dia ainda não foi cadastrado.
            </p>
          </div>
        `;
        return;
      }

      container.innerHTML = itens.map(item => `
        <div class="menu-card bg-white rounded-lg p-6 shadow-md" style="border-top: 4px solid #ff6b35;">
          <h3 class="text-lg font-bold text-gray-800">${escapeHtml(item.name)}</h3>
          <p class="text-gray-500 text-sm mt-2">${escapeHtml(item.desc || '')}</p>

          <div class="mt-4">
            ${(item.sizes || []).map((s, idx) => `
              <button
                type="button"
                onclick="abrirDetalhes('${escapeHtml(item.id)}', ${idx})"
                class="w-full mb-2 px-4 py-2 rounded-lg font-semibold text-white transition transform hover:scale-105"
                style="background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%);"
              >
                ${escapeHtml(s.size)} - R$ ${formatBRL(s.price)}
              </button>
            `).join('')}
          </div>
        </div>
      `).join('');
    }

    /*********************
     * MODAL: PEDIDO
     *********************/
    function cancelarPedido() {
      const modal = document.getElementById('modal-detalhes');
      if (modal) modal.remove();
    }

    function alterarQtd(delta) {
      const el = document.getElementById("qtd");
      if (!el) return;
      let v = parseInt(String(el.value || "1").replace(/\D/g, ""), 10);
      if (!Number.isFinite(v) || v < 1) v = 1;
      v = Math.max(1, v + delta);
      el.value = String(v);
      updateTotalPreview();
    }

    function updateTotalPreview() {
      const price = Number(document.getElementById('modal-price')?.value || 0);
      const qty = Math.max(1, parseInt((document.getElementById('qtd')?.value || '1').replace(/\D/g,''), 10) || 1);
      const el = document.getElementById('total-preview');
      if (el) el.textContent = toBRL(price * qty);
    }

    function abrirDetalhes(itemId, sizeIdx) {
      const itens = menuSemana[diaSelecionado] || [];
      const item = itens.find(m => String(m.id) === String(itemId));
      if (!item) return;

      const size = (item.sizes || [])[sizeIdx];
      if (!size) return;

      cancelarPedido();

      const c = loadCustomer();

      const modal = document.createElement('div');
      modal.id = 'modal-detalhes';
      modal.style.cssText = `
        position: fixed; inset: 0;
        background: rgba(0,0,0,0.5);
        display: flex; align-items: center; justify-content: center;
        z-index: 1000;
        padding: 16px;
      `;

      modal.innerHTML = `
        <div role="dialog" aria-modal="true" aria-label="Detalhes do pedido"
             style="background: white; border-radius: 12px; padding: 24px; max-width: 520px; width: 100%; max-height: 90vh; overflow-y: auto;">
          <div style="display:flex; align-items:start; justify-content:space-between; gap:12px;">
            <h2 style="font-size: 20px; font-weight: bold; color: #333; margin-bottom: 16px;">
              ${escapeHtml(item.name)}
            </h2>
            <button type="button" onclick="cancelarPedido()" aria-label="Fechar"
              style="border:none; background:transparent; font-size:22px; line-height:22px; cursor:pointer; color:#666;">×</button>
          </div>

          <input type="hidden" id="modal-itemId" value="${escapeHtml(itemId)}">
          <input type="hidden" id="modal-sizeIdx" value="${sizeIdx}">
          <input type="hidden" id="modal-price" value="${escapeHtml(size.price)}">

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">📍 Tamanho:</label>
            <div style="padding: 8px 12px; background: #f5f5f5; border-radius: 6px; color: #666;">
              ${escapeHtml(size.size)} - R$ ${formatBRL(size.price)}
            </div>
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">🧮 Quantidade:</label>
            <div style="display:flex;align-items:center;gap:10px;">
              <button type="button" onclick="alterarQtd(-1)"
                style="width:44px;height:40px;border-radius:10px;background:#eee;border:none;font-size:18px;cursor:pointer;">−</button>

              <input id="qtd" value="1" inputmode="numeric"
                oninput="this.value=this.value.replace(/\\D/g,''); if(!this.value) this.value='1'; updateTotalPreview();"
                style="width:80px;text-align:center;padding:10px;border:1px solid #ddd;border-radius:10px;font-size:14px;" />

              <button type="button" onclick="alterarQtd(1)"
                style="width:44px;height:40px;border-radius:10px;background:#111827;color:#fff;border:none;font-size:18px;cursor:pointer;">+</button>
            </div>
            <div style="margin-top:10px;color:#111827;font-weight:bold;">
              Total: <span id="total-preview">${toBRL(Number(size.price))}</span>
            </div>
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">👤 Seu Nome:</label>
            <input type="text" id="cliente-nome" placeholder="Digite seu nome completo"
              value="${escapeHtml(c.nome || '')}"
              style="width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 14px;">
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">📱 Seu Telefone:</label>
            <input type="tel" id="cliente-telefone" placeholder="(31) 99999-9999"
              value="${escapeHtml(c.telefone || '')}"
              style="width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 14px;">
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">📍 Seu Endereço:</label>
            <input type="text" id="cliente-endereco" placeholder="Rua, número, complemento"
              value="${escapeHtml(c.endereco || '')}"
              style="width: 100%; padding: 10px; border: 1px solid #ddd; border-radius: 6px; font-size: 14px;">
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">
              🥗 Escolha os acompanhamentos ${currentConfig.min_acompanhamentos > 0 ? `(mín: ${currentConfig.min_acompanhamentos})` : ''}
            </label>
            <div id="acompanhamento-list" style="max-height: 150px; overflow-y: auto; padding-right: 6px;">
              ${acompanhamentos.map(acomp => `
                <label style="display: flex; align-items: center; padding: 8px 0; cursor: pointer; color: #666;">
                  <input type="checkbox" class="acomp-checkbox" value="${escapeHtml(acomp)}"
                    style="margin-right: 8px; width: 18px; height: 18px;">
                  ${escapeHtml(acomp)}
                </label>
              `).join('')}
            </div>
          </div>

          <div style="margin-bottom: 16px;">
            <label style="display: block; font-weight: bold; color: #333; margin-bottom: 8px;">📝 Observações:</label>
            <textarea id="observacoes" placeholder="Ex: Sem cebola, sem sal..."
              style="width: 100%; padding: 8px; border: 1px solid #ddd; border-radius: 6px; font-family: Arial; font-size: 14px; min-height: 60px;"></textarea>
          </div>

          <div style="display: flex; gap: 10px; flex-wrap: wrap;">
            <button type="button" onclick="cancelarPedido()" class="flex-1 px-4 py-2 rounded-lg font-semibold"
              style="min-width: 140px; background: #ddd; color: #333; border: none; cursor: pointer;">
              Cancelar
            </button>

            <button type="button" onclick="copiarPedido()" class="flex-1 px-4 py-2 rounded-lg font-semibold"
              style="min-width: 140px; background: #111827; color: #fff; border: none; cursor: pointer;">
              📋 Copiar pedido
            </button>

            <button type="button" onclick="finalizarPedido()" class="flex-1 px-4 py-2 rounded-lg font-semibold text-white"
              style="min-width: 180px; background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%); border: none; cursor: pointer;">
              ✅ Pedir no WhatsApp
            </button>
          </div>

          <p style="margin-top: 14px; font-size: 12px; color:#777;">
            Dica: se o WhatsApp não abrir, use “Copiar pedido” e cole manualmente.
          </p>
        </div>
      `;

      document.body.appendChild(modal);

      modal.addEventListener('click', (e) => {
        if (e.target === modal) cancelarPedido();
      });

      function escHandler(e) {
        if (e.key === 'Escape') {
          cancelarPedido();
          document.removeEventListener('keydown', escHandler);
        }
      }
      document.addEventListener('keydown', escHandler);

      setTimeout(() => {
        const nome = document.getElementById('cliente-nome');
        if (nome) nome.focus();
      }, 50);

      updateTotalPreview();
    }

    function getFormDataOrAlert() {
      const clienteNome = (document.getElementById('cliente-nome')?.value || '').trim();
      const clienteTelefone = (document.getElementById('cliente-telefone')?.value || '').trim();
      const clienteEndereco = (document.getElementById('cliente-endereco')?.value || '').trim();

      const qty = Math.max(1, parseInt((document.getElementById('qtd')?.value || '1').replace(/\D/g,''), 10) || 1);

      if (!clienteNome) { alert('⚠️ Por favor, digite seu nome!'); return null; }
      if (!clienteTelefone) { alert('⚠️ Por favor, digite seu telefone!'); return null; }
      if (!clienteEndereco) { alert('⚠️ Por favor, digite seu endereço!'); return null; }

      // salva dados do cliente
      saveCustomer({ nome: clienteNome, telefone: clienteTelefone, endereco: clienteEndereco });

      const checkboxes = document.querySelectorAll('.acomp-checkbox:checked');
      const acompSelecionados = Array.from(checkboxes).map(cb => cb.value);

      if (currentConfig.min_acompanhamentos > 0 && acompSelecionados.length < currentConfig.min_acompanhamentos) {
        alert(`⚠️ Escolha pelo menos ${currentConfig.min_acompanhamentos} acompanhamentos!`);
        return null;
      }

      const observacoes = (document.getElementById('observacoes')?.value || '').trim();

      return { clienteNome, clienteTelefone, clienteEndereco, acompSelecionados, observacoes, qty };
    }

    function buildOrderMessage(item, size, clienteNome, clienteTelefone, clienteEndereco, acompSelecionados, observacoes, qty) {
      const total = Number(size.price) * Number(qty || 1);

      let message = `🍱 *NOVO PEDIDO*\n\n`;
      message += `👤 *Cliente:* ${clienteNome}\n`;
      message += `📱 *Telefone:* ${clienteTelefone}\n`;
      message += `📍 *Endereço:* ${clienteEndereco}\n\n`;
      message += `━━━━━━━━━━━━━━━━━━\n\n`;
      message += `📌 *Prato:* ${item.name}\n`;
      message += `📏 *Tamanho:* ${size.size}\n`;
      message += `🧮 *Quantidade:* ${qty}\n`;
      message += `💰 *Unitário:* R$ ${formatBRL(size.price)}\n`;
      message += `💵 *Total:* R$ ${formatBRL(total)}\n\n`;

      if (acompSelecionados.length) {
        message += `🥗 *Acompanhamentos:*\n${acompSelecionados.join('\n')}\n\n`;
      }
      if (observacoes) {
        message += `📝 *Observações:*\n${observacoes}\n`;
      }
      return message;
    }

    async function copiarPedido() {
      const itemId = document.getElementById('modal-itemId')?.value;
      const sizeIdx = Number(document.getElementById('modal-sizeIdx')?.value || 0);

      const itens = menuSemana[diaSelecionado] || [];
      const item = itens.find(m => String(m.id) === String(itemId));
      if (!item) return;

      const size = (item.sizes || [])[sizeIdx];
      if (!size) return;

      const form = getFormDataOrAlert();
      if (!form) return;

      const msg = buildOrderMessage(
        item, size,
        form.clienteNome,
        form.clienteTelefone,
        form.clienteEndereco,
        form.acompSelecionados,
        form.observacoes,
        form.qty
      );

      await copyToClipboard(msg);
      // (Opcional) Se quiser contar "Copiar" como venda, descomente:
      // registerSale(item, size, form);
    }

    function registerSale(item, size, form) {
      const total = Number(size.price) * Number(form.qty || 1);
      addSale({
        dayKey: todayKey(),
        ts: Date.now(),
        clienteNome: form.clienteNome,
        clienteTelefone: form.clienteTelefone,
        clienteEndereco: form.clienteEndereco,
        itemId: item.id,
        itemName: item.name,
        size: size.size,
        price: Number(size.price),
        qty: Number(form.qty || 1),
        total,
        observacoes: form.observacoes || "",
      });
    }

    function finalizarPedido() {
      const itemId = document.getElementById('modal-itemId')?.value;
      const sizeIdx = Number(document.getElementById('modal-sizeIdx')?.value || 0);

      const itens = menuSemana[diaSelecionado] || [];
      const item = itens.find(m => String(m.id) === String(itemId));
      if (!item) return;

      const size = (item.sizes || [])[sizeIdx];
      if (!size) return;

      const form = getFormDataOrAlert();
      if (!form) return;

      const whatsappNumber = normalizeWhatsappNumber(currentConfig.whatsapp_number || defaultConfig.whatsapp_number);
      if (!whatsappNumber) {
        alert('⚠️ WhatsApp do restaurante não configurado!');
        return;
      }

      // registra venda (conta quando clica "Pedir no WhatsApp")
      registerSale(item, size, form);

      const message = buildOrderMessage(
        item, size,
        form.clienteNome,
        form.clienteTelefone,
        form.clienteEndereco,
        form.acompSelecionados,
        form.observacoes,
        form.qty
      );

      const encoded = encodeURIComponent(message);
      const whatsappUrl = `https://wa.me/${whatsappNumber}?text=${encoded}`;

      cancelarPedido();
      setTimeout(() => window.open(whatsappUrl, '_blank'), 100);
    }

    /*********************
     * MÁSCARA TELEFONE
     *********************/
    document.addEventListener('input', (e) => {
      if (e.target && e.target.id === 'cliente-telefone') {
        let v = e.target.value.replace(/\D/g, '').slice(0, 11);
        if (v.length >= 2) v = `(${v.slice(0, 2)}) ${v.slice(2)}`;
        if (v.replace(/\D/g,'').length >= 11) {
          const digits = v.replace(/\D/g,'');
          v = `(${digits.slice(0,2)}) ${digits.slice(2,7)}-${digits.slice(7,11)}`;
        }
        e.target.value = v;
      }
    });

    /*********************
     * ADMIN: RENDER + EDITOR
     *********************/
    function adminRow(it, idx) {
      const sizes = Array.isArray(it.sizes) ? it.sizes : [];
      const small = sizes[0] || { size: "Pequeno", price: "" };
      const large = sizes[1] || { size: "Grande", price: "" };

      return `
        <div class="p-4 rounded-lg border border-gray-200">
          <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
            <div>
              <label class="text-xs text-gray-500">Nome</label>
              <input class="w-full mt-1 p-2 border rounded" data-idx="${idx}" data-field="name"
                value="${escapeHtml(it.name || '')}" placeholder="Ex: Frango com quiabo" />
            </div>
            <div>
              <label class="text-xs text-gray-500">Descrição</label>
              <input class="w-full mt-1 p-2 border rounded" data-idx="${idx}" data-field="desc"
                value="${escapeHtml(it.desc || '')}" placeholder="Ex: Arroz, feijão,..." />
            </div>
            <div>
              <label class="text-xs text-gray-500">Preço Pequeno</label>
              <input class="w-full mt-1 p-2 border rounded" data-idx="${idx}" data-field="price_small"
                value="${escapeHtml(String(small.price ?? ''))}" placeholder="Ex: 18" />
              <label class="text-xs text-gray-500 mt-2 block">Preço Grande</label>
              <input class="w-full mt-1 p-2 border rounded" data-idx="${idx}" data-field="price_large"
                value="${escapeHtml(String(large.price ?? ''))}" placeholder="Ex: 20" />
            </div>
          </div>

          <div class="mt-3 flex justify-end">
            <button onclick="removeItemAdmin(${idx})" class="px-3 py-2 rounded font-semibold"
              style="background:#fee2e2;color:#991b1b;">🗑 Remover</button>
          </div>
        </div>
      `;
    }

    function addItemAdmin() {
      menuSemana[diaSelecionado] = menuSemana[diaSelecionado] || [];
      menuSemana[diaSelecionado].push({
        id: crypto?.randomUUID ? crypto.randomUUID() : String(Date.now()),
        name: "",
        desc: "",
        sizes: [
          { size: "Pequeno", price: "" },
          { size: "Grande", price: "" },
        ]
      });
      renderAdmin();
    }

    function removeItemAdmin(idx) {
      menuSemana[diaSelecionado].splice(idx, 1);
      renderAdmin();
    }

    function saveAdmin() {
      // lê inputs
      const inputs = document.querySelectorAll('#menu-container [data-idx][data-field]');
      inputs.forEach(inp => {
        const idx = Number(inp.getAttribute('data-idx'));
        const field = inp.getAttribute('data-field');
        const val = (inp.value || '').trim();

        const it = menuSemana[diaSelecionado][idx];

        if (field === 'name') it.name = val;
        if (field === 'desc') it.desc = val;

        if (field === 'price_small') it.sizes[0].price = String(val).replace(',', '.');
        if (field === 'price_large') it.sizes[1].price = String(val).replace(',', '.');
      });

      // limpa: mantém só itens com nome + preços numéricos (se vazios, fica vazio mesmo)
      menuSemana[diaSelecionado] = (menuSemana[diaSelecionado] || [])
        .map(it => {
          const p1 = Number(String(it.sizes?.[0]?.price ?? '').replace(',', '.'));
          const p2 = Number(String(it.sizes?.[1]?.price ?? '').replace(',', '.'));
          return {
            ...it,
            name: (it.name || '').trim(),
            desc: (it.desc || '').trim(),
            sizes: [
              { size: "Pequeno", price: Number.isFinite(p1) ? p1 : 0 },
              { size: "Grande", price: Number.isFinite(p2) ? p2 : 0 },
            ]
          };
        })
        .filter(it => it.name);

      saveMenuSemana(menuSemana);
      alert("✅ Cardápio salvo!");
      renderAdmin();
    }

    function renderAdmin() {
      if (!requireAdminLogin()) return;

      const container = document.getElementById('menu-container');
      const titulo = document.getElementById('titulo-dia');
      const subtitulo = document.getElementById('subtitulo-dia');

      const diaObj = diasSemana.find(d => d.key === diaSelecionado);
      titulo.textContent = `ADM — editar ${diaObj ? diaObj.label : ''}`;
      subtitulo.textContent = 'Cadastre os itens e veja o faturamento de hoje.';

      const t = totalsOfToday();
      const vendasHoje = salesOfToday().slice().reverse();

      const itens = menuSemana[diaSelecionado] || [];

      container.innerHTML = `
        <div class="bg-white rounded-lg p-6 shadow-md col-span-full" style="border-top: 4px solid #111827;">
          <div class="flex items-center justify-between flex-wrap gap-3 mb-4">
            <div class="font-bold text-gray-800">Painel do ADM</div>
            <div class="flex gap-2 flex-wrap">
              <button onclick="addItemAdmin()" class="px-4 py-2 rounded-lg font-semibold text-white"
                style="background:#111827;">+ Adicionar item</button>
              <button onclick="saveAdmin()" class="px-4 py-2 rounded-lg font-semibold text-white"
                style="background: linear-gradient(135deg, #ff6b35 0%, #f7931e 100%);">💾 Salvar cardápio</button>
              <button onclick="sairAdmin()" class="px-4 py-2 rounded-lg font-semibold"
                style="background:#eee;color:#333;">Sair</button>
            </div>
          </div>

          <div class="grid grid-cols-1 md:grid-cols-3 gap-3 mb-4">
            <div class="p-4 rounded-lg border border-gray-200">
              <div class="text-xs text-gray-500">Faturamento de hoje</div>
              <div class="text-2xl font-bold text-gray-900">${toBRL(t.total)}</div>
            </div>
            <div class="p-4 rounded-lg border border-gray-200">
              <div class="text-xs text-gray-500">Pedidos hoje</div>
              <div class="text-2xl font-bold text-gray-900">${t.count}</div>
            </div>
            <div class="p-4 rounded-lg border border-gray-200">
              <div class="text-xs text-gray-500">Ticket médio</div>
              <div class="text-2xl font-bold text-gray-900">${toBRL(t.avg)}</div>
            </div>
          </div>

          <div class="flex gap-2 flex-wrap mb-6">
            <button onclick="exportTodayCSV()" class="px-4 py-2 rounded-lg font-semibold text-white" style="background:#0f766e;">
              ⬇️ Exportar CSV (hoje)
            </button>
            <button onclick="if(confirm('Limpar faturamento de hoje?')){ clearTodaySales(); renderAdmin(); }"
              class="px-4 py-2 rounded-lg font-semibold" style="background:#fee2e2;color:#991b1b;">
              🧹 Limpar hoje
            </button>
          </div>

          <div class="p-4 rounded-lg border border-gray-200 mb-6">
            <div class="font-bold text-gray-800 mb-2">Pedidos de hoje</div>
            <div class="text-sm text-gray-700">
              ${vendasHoje.length ? vendasHoje.map(s => `
                <div style="padding:10px 0;border-bottom:1px solid #eee;">
                  <div><b>${escapeHtml(s.clienteNome || '')}</b> — ${escapeHtml(s.itemName || '')} (${escapeHtml(s.size || '')}) x${Number(s.qty||1)}</div>
                  <div style="color:#555;margin-top:4px;">Total: <b>${toBRL(s.total)}</b> — ${new Date(s.ts).toLocaleTimeString('pt-BR')}</div>
                </div>
              `).join('') : `<span class="text-gray-500">Nenhum pedido registrado hoje.</span>`}
            </div>
          </div>

          <div class="font-bold text-gray-800 mb-3">Itens do dia (${escapeHtml(diaObj?.label || '')})</div>
          <div id="admin-list" class="space-y-3">
            ${itens.map((it, idx) => adminRow(it, idx)).join('')}
          </div>

          ${!itens.length ? `<p class="text-sm text-gray-500 mt-3">Nenhum item ainda. Clique em “Adicionar item”.</p>` : ``}
        </div>
      `;
    }

    /*********************
     * START / APP
     *********************/
    function startApp() {
      // header
      const nameEl = document.getElementById('restaurant-name');
      if (nameEl) nameEl.textContent = currentConfig.restaurant_name || defaultConfig.restaurant_name;

      const adminHint = document.getElementById('admin-hint');
      if (adminHint) adminHint.style.display = isAdminMode() ? 'block' : 'none';

      renderTabs();
      if (isAdminMode()) renderAdmin();
      else renderMenuDoDia();
    }

    window.addEventListener('hashchange', startApp);

    /*********************
     * ELEMENT SDK (CONFIG)
     *********************/
    async function onConfigChange(config) {
      currentConfig = { ...defaultConfig, ...config };
      const nameEl = document.getElementById('restaurant-name');
      if (nameEl) nameEl.textContent = currentConfig.restaurant_name || defaultConfig.restaurant_name;
    }

    function mapToCapabilities() {
      return { recolorables: [], borderables: [], fontEditable: undefined, fontSizeable: undefined };
    }

    function mapToEditPanelValues(config) {
      const cfg = { ...defaultConfig, ...config };
      return new Map([
        ['restaurant_name', cfg.restaurant_name],
        ['whatsapp_number', cfg.whatsapp_number],
        ['min_acompanhamentos', String(cfg.min_acompanhamentos ?? 0)]
      ]);
    }

    if (window.elementSdk) {
      window.elementSdk.init({ defaultConfig, onConfigChange, mapToCapabilities, mapToEditPanelValues });
    }

    // inicia
    startApp();
  </script>
</body>
</html>
