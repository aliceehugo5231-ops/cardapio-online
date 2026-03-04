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

<div class="flex gap-2 flex-wrap mb-4">
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
  <div class="text-sm text-gray-600" id="lista-pedidos-hoje">
    ${salesOfToday().slice().reverse().map(s => `
      <div style="padding:10px 0;border-bottom:1px solid #eee;">
        <div><b>${escapeHtml(s.clienteNome || '')}</b> — ${escapeHtml(s.itemName || '')} (${escapeHtml(s.size || '')}) x${s.qty}</div>
        <div style="color:#555;margin-top:4px;">Total: <b>${toBRL(s.total)}</b> — ${new Date(s.ts).toLocaleTimeString('pt-BR')}</div>
      </div>
    `).join('') || '<span class="text-gray-500">Nenhum pedido registrado hoje.</span>'}
  </div>
</div>
