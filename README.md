# Merval

<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Merval en Tiempo Real</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: 'Segoe UI', sans-serif;
      background: #0d1117;
      color: #e6edf3;
      padding: 24px;
    }
    h1 { font-size: 1.4rem; margin-bottom: 4px; color: #58a6ff; }
    .sub { font-size: 0.8rem; color: #8b949e; margin-bottom: 20px; }
    #ultima-actualizacion { font-size: 0.75rem; color: #8b949e; margin-bottom: 16px; }
    table { width: 100%; border-collapse: collapse; font-size: 0.88rem; }
    thead th {
      background: #161b22;
      padding: 10px 14px;
      text-align: left;
      color: #8b949e;
      font-weight: 600;
      border-bottom: 1px solid #30363d;
      cursor: pointer;
    }
    thead th:hover { color: #58a6ff; }
    tbody tr:hover { background: #161b22; }
    td { padding: 9px 14px; border-bottom: 1px solid #21262d; }
    .ticker { font-weight: 700; color: #58a6ff; }
    .pos { color: #3fb950; }
    .neg { color: #f85149; }
    .neu { color: #8b949e; }
    #estado { text-align: center; padding: 40px; color: #8b949e; }
    .badge {
      display: inline-block;
      padding: 2px 8px;
      border-radius: 4px;
      font-size: 0.75rem;
      font-weight: 600;
    }
    .badge.pos { background: #1a3a1f; color: #3fb950; }
    .badge.neg { background: #3a1a1a; color: #f85149; }
    .badge.neu { background: #1c2128; color: #8b949e; }
  </style>
</head>
<body>
  <h1>📈 Merval — Cotizaciones en Tiempo Real</h1>
  <p class="sub">Acciones del S&P Merval · Moneda ARS · Plazo 24hs · Fuente: data912.com</p>
  <div id="ultima-actualizacion">Cargando...</div>
  <div id="estado">⏳ Obteniendo datos...</div>
  <table id="tabla" style="display:none">
    <thead>
      <tr>
        <th onclick="sortBy('ticker')">Ticker</th>
        <th onclick="sortBy('nombre')">Empresa</th>
        <th onclick="sortBy('ultimo')">Último (ARS)</th>
        <th onclick="sortBy('variacion')">Var. %</th>
        <th onclick="sortBy('apertura')">Apertura</th>
        <th onclick="sortBy('max')">Máx.</th>
        <th onclick="sortBy('min')">Mín.</th>
        <th onclick="sortBy('volumen')">Volumen</th>
      </tr>
    </thead>
    <tbody id="body"></tbody>
  </table>

  <script>
    // Tickers del S&P Merval
    const MERVAL = [
      'ALUA','BBAR','BMA','BYMA','CECO2','CEPU','COME',
      'CRES','CVH','GGAL','HARG','LOMA','METR','MIRG',
      'PAMP','SUPV','TECO2','TGNO4','TGSU2','TRAN','TXAR',
      'VALO','YPFD'
    ];

    const NOMBRES = {
      ALUA:'Aluar', BBAR:'Banco BBVA', BMA:'Banco Macro',
      BYMA:'BYMA', CECO2:'Cen.Cost.', CEPU:'CEPU',
      COME:'Sociedad Comercial del Plata', CRES:'Cresud',
      CVH:'Cablevision', GGAL:'Grupo Financiero Galicia',
      HARG:'Holcim', LOMA:'Loma Negra', METR:'Metrogas',
      MIRG:'Mirgor', PAMP:'Pampa Energía', SUPV:'Supervielle',
      TECO2:'Telecom', TGNO4:'TGN', TGSU2:'TGS', TRAN:'Transener',
      TXAR:'Ternium', VALO:'Grupo Valores', YPFD:'YPF'
    };

    let datos = [];
    let sortDir = {};

    async function fetchTicker(ticker) {
      try {
        const r = await fetch(
          `https://data912.com/api/v1/quotes/${ticker}?settlement=24hs`,
          { headers: { 'Accept': 'application/json' } }
        );
        if (!r.ok) return null;
        return await r.json();
      } catch { return null; }
    }

    function clase(v) {
      if (v > 0) return 'pos';
      if (v < 0) return 'neg';
      return 'neu';
    }

    function fmt(n, dec=2) {
      if (n == null || isNaN(n)) return '—';
      return Number(n).toLocaleString('es-AR', {
        minimumFractionDigits: dec,
        maximumFractionDigits: dec
      });
    }

    function fmtVol(n) {
      if (!n) return '—';
      if (n >= 1e6) return (n/1e6).toFixed(1) + 'M';
      if (n >= 1e3) return (n/1e3).toFixed(0) + 'K';
      return n;
    }

    function render() {
      const tbody = document.getElementById('body');
      tbody.innerHTML = datos.map(d => {
        const vp = parseFloat(d.variacion) || 0;
        const c = clase(vp);
        return `<tr>
          <td class="ticker">${d.ticker}</td>
          <td>${NOMBRES[d.ticker] || d.ticker}</td>
          <td>${fmt(d.ultimo)}</td>
          <td><span class="badge ${c}">${vp > 0 ? '+' : ''}${fmt(vp)}%</span></td>
          <td>${fmt(d.apertura)}</td>
          <td>${fmt(d.max)}</td>
          <td>${fmt(d.min)}</td>
          <td>${fmtVol(d.volumen)}</td>
        </tr>`;
      }).join('');
    }

    function sortBy(campo) {
      sortDir[campo] = !sortDir[campo];
      datos.sort((a, b) => {
        const va = parseFloat(a[campo]) || a[campo] || '';
        const vb = parseFloat(b[campo]) || b[campo] || '';
        return sortDir[campo]
          ? (va > vb ? 1 : -1)
          : (va < vb ? 1 : -1);
      });
      render();
    }

    async function cargarTodos() {
      document.getElementById('estado').textContent = '⏳ Cargando cotizaciones...';
      // Fetch en paralelo
      const resultados = await Promise.all(MERVAL.map(t => fetchTicker(t)));
      datos = resultados
        .map((r, i) => {
          if (!r) return null;
          // Adaptá según la estructura real que devuelve la API
          const q = Array.isArray(r) ? r[0] : r;
          return {
            ticker: MERVAL[i],
            ultimo:   q?.last ?? q?.c ?? q?.close ?? null,
            variacion:q?.pct_change ?? q?.change_pct ?? q?.variacion ?? null,
            apertura: q?.open ?? q?.o ?? null,
            max:      q?.high ?? q?.h ?? null,
            min:      q?.low ?? q?.l ?? null,
            volumen:  q?.volume ?? q?.v ?? null,
          };
        })
        .filter(Boolean);

      document.getElementById('estado').style.display = 'none';
      document.getElementById('tabla').style.display = 'table';
      document.getElementById('ultima-actualizacion').textContent =
        '🕐 Última actualización: ' + new Date().toLocaleTimeString('es-AR');
      render();
    }

    cargarTodos();
    // Auto-refresh cada 60 segundos
    setInterval(cargarTodos, 60000);
  </script>
</body>
</html>
