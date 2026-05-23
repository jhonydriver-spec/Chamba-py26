# Chamba-py26
Ofrecer y buscar trabajo 
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ChampaPy</title>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; font-family: 'Segoe UI', Arial, sans-serif; }
body { background: #f8fafc; padding-bottom: 80px; }
header { background: #2563eb; color: white; padding: 15px; text-align: center; }
.container { padding: 15px; max-width: 700px; margin: 0 auto; }
.btn { width: 100%; padding: 12px; background: #2563eb; border: none; border-radius: 8px; color: white; font-weight: bold; cursor: pointer; margin-top: 10px; }
.btn-destacado { background: #f59e0b; }
.btn-danger { background: #ef4444; }
input, select, textarea { width: 100%; padding: 10px; margin: 8px 0; border: 1px solid #cbd5e1; border-radius: 8px; }
.card { background: white; padding: 15px; border-radius: 12px; margin-bottom: 12px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
.grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
.grid img,.grid video { width: 100%; height: 150px; object-fit: cover; border-radius: 8px; }
.badge { padding: 2px 8px; border-radius: 12px; font-size: 12px; color: white; }
.tab-nav { position: fixed; bottom: 0; left: 0; width: 100%; background: white; display: flex; border-top: 1px solid #e2e8f0; }
.tab-btn { flex: 1; padding: 12px; background: none; border: none; cursor: pointer; }
.tab-btn.active { color: #2563eb; font-weight: bold; }
.hidden { display: none; }
details { margin-top: 10px; }
summary { cursor: pointer; color: #2563eb; }
</style>
</head>
<body>

<div id="app">
<header><h1>ChampaPy</h1><small>Trabajadores verificados cerca tuyo</small></header>

<div class="container">
  <div id="pantalla-buscar">
    <input type="text" id="buscar-oficio" placeholder="Buscar: electricista, plomero..." oninput="renderizarTrabajadores()">
    <select id="filtro-zona" onchange="renderizarTrabajadores()">
      <option value="">Todas las zonas</option>
      <option>Asunción</option><option>Luque</option><option>San Lorenzo</option>
    </select>
    <div id="lista-trabajadores"></div>
  </div>

  <div id="pantalla-perfil" class="hidden">
    <div id="registro-form">
      <h3>Crear tu perfil gratis</h3>
      <input id="nombre" placeholder="Nombre completo">
      <input id="telefono" placeholder="0981 123 456" type="tel">
      <select id="oficio">
        <option>Electricista</option><option>Plomero</option><option>Pintor</option><option>Albañil</option>
      </select>
      <input id="zona" placeholder="Tu zona">
      <textarea id="descripcion" placeholder="Contá qué hacés" rows="3"></textarea>
      <button class="btn" onclick="registrarTrabajador()">Crear perfil gratis</button>
      <p style="font-size:12px; color:#64748b;">Verificación gratis. Destacado Gs 5.000/mes</p>
    </div>

    <div id="perfil-creado" class="hidden">
      <h3>Mis trabajos</h3>
      <input type="file" id="archivo" accept="image/*,video/*" multiple>
      <button class="btn" onclick="subirTrabajo()">Subir foto/video</button>
      <div id="mis-trabajos"></div>
      <div id="seccion-creditos"></div>
    </div>
  </div>
</div>

<div class="tab-nav">
  <button class="tab-btn active" onclick="cambiarTab('buscar', event)">Buscar</button>
  <button class="tab-btn" onclick="cambiarTab('perfil', event)">Mi Perfil</button>
</div>
</div>

<div id="admin" class="hidden"></div>

<script>
const CLAVE_ADMIN = "RealMadrid2026"; // CAMBIÁ ESTO
let trabajadores = JSON.parse(localStorage.getItem('champa_trabajadores')) || [];
let usuarioActual = JSON.parse(localStorage.getItem('champa_usuario')) || null;

// Check si es admin por URL
if (window.location.search.includes('admin=' + CLAVE_ADMIN)) {
  mostrarAdmin();
} else {
  limpiarDestacados();
  renderizarTrabajadores();
  checkUsuario();
}

function registrarTrabajador() {
  let t = {
    id: 'trab_' + Date.now(), nombre: document.getElementById('nombre').value,
    telefono: document.getElementById('telefono').value, oficio: document.getElementById('oficio').value,
    zona: document.getElementById('zona').value, descripcion: document.getElementById('descripcion').value,
    trabajos: [], reseñas: [], rating_promedio: 0, creditos: 0, verificado: false,
    destacado: false, destacado_hasta: null
  };
  if (!t.nombre ||!t.telefono) return alert('Completá nombre y teléfono');
  trabajadores.push(t); usuarioActual = t; guardarDatos(); checkUsuario();
}

function subirTrabajo() {
  let archivo = document.getElementById('archivo').files[0]; if (!archivo) return;
  let reader = new FileReader();
  reader.onload = function(e) {
    usuarioActual.trabajos.push({url: e.target.result, tipo: archivo.type.startsWith('video')? 'video' : 'imagen'});
    let idx = trabajadores.findIndex(t => t.id === usuarioActual.id);
    trabajadores[idx] = usuarioActual; guardarDatos(); renderizarMisTrabajos();
  };
  reader.readAsDataURL(archivo);
}

function renderizarTrabajadores() {
  let filtroOficio = document.getElementById('buscar-oficio').value.toLowerCase();
  let filtroZona = document.getElementById('filtro-zona').value;
  let filtrados = trabajadores.filter(t =>
    t.oficio.toLowerCase().includes(filtroOficio) && (filtroZona === '' || t.zona === filtroZona)
  ).sort((a, b) => {
    if (a.destacado &&!b.destacado) return -1;
    return b.rating_promedio - a.rating_promedio;
  });

  document.getElementById('lista-trabajadores').innerHTML = filtrados.map(t => `
    <div class="card">
      <h3>${t.nombre} ${t.verificado? '<span class="badge" style="background:#22c55e;">✓</span>' : ''}
          ${t.destacado? '<span class="badge" style="background:#f59e0b;">★</span>' : ''}</h3>
      <p>${t.oficio} - ${t.zona} | ⭐ ${t.rating_promedio.toFixed(1)} (${t.reseñas.length})</p>
      <p>${t.descripcion}</p>
      <div class="grid">${t.trabajos.slice(0,4).map(tr =>
        tr.tipo === 'video'? `<video src="${tr.url}" controls></video>` : `<img src="${tr.url}">`
      ).join('')}</div>
      <a href="https://wa.me/595${t.telefono.replace(/\D/g,'')}" class="btn" style="text-decoration:none; display:block; text-align:center;">Contactar WhatsApp</a>
      <details>
        <summary>Dejar reseña</summary>
        <select id="estrellas-${t.id}"><option value="5">5</option><option value="4">4</option><option value="3">3</option></select>
        <input id="comentario-${t.id}" placeholder="Comentario">
        <button class="btn" onclick="dejarReseña('${t.id}')">Enviar</button>
      </details>
    </div>
  `).join('');
}

function dejarReseña(id) {
  let estrellas = parseInt(document.getElementById(`estrellas-${id}`).value);
  let comentario = document.getElementById(`comentario-${id}`).value;
  if (!comentario) return;
  let t = trabajadores.find(x => x.id === id);
  t.reseñas.push({estrellas, comentario});
  t.rating_promedio = t.reseñas.reduce((sum, r) => sum + r.estrellas, 0) / t.reseñas.length;
  guardarDatos(); renderizarTrabajadores();
}

function renderizarMisTrabajos() {
  document.getElementById('mis-trabajos').innerHTML = `
    <div class="grid">${usuarioActual.trabajos.map(t =>
      t.tipo === 'video'? `<video src="${t.url}" controls></video>` : `<img src="${t.url}">`
    ).join('')}</div>
  `;
  document.getElementById('seccion-creditos').innerHTML = `
    <div style="margin-top:20px; padding:15px; background:#fef3c7; border-radius:8px;">
      <strong>Créditos: ${usuarioActual.creditos || 0}</strong>
      <p>1 crédito = Gs 5.000 = 30 días destacado</p>
      <p>Pagá por billetera personal 0976-544-936 y enviá captura</p>
      <button class="btn btn-destacado" onclick="usarCredito()">Activar 30 días</button>
    </div>
  `;
}

function usarCredito() {
  if ((usuarioActual.creditos || 0) <= 0) return alert('No tenés créditos');
  usuarioActual.creditos--;
  let fecha = new Date(); fecha.setDate(fecha.getDate() + 30);
  usuarioActual.destacado = true; usuarioActual.destacado_hasta = fecha.toISOString();
  let idx = trabajadores.findIndex(t => t.id === usuarioActual.id);
  trabajadores[idx] = usuarioActual; guardarDatos(); renderizarMisTrabajos();
  alert('Destacado activado 30 días');
}

function checkUsuario() {
  if (usuarioActual) {
    document.getElementById('registro-form').classList.add('hidden');
    document.getElementById('perfil-creado').classList.remove('hidden');
    renderizarMisTrabajos();
  }
}

function cambiarTab(tab, e) {
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  e.target.classList.add('active');
  document.getElementById('pantalla-buscar').classList.toggle('hidden', tab!== 'buscar');
  document.getElementById('pantalla-perfil').classList.toggle('hidden', tab!== 'perfil');
}

function limpiarDestacados() {
  let ahora = new Date().toISOString();
  trabajadores.forEach(t => { if (t.destacado && t.destacado_hasta < ahora) t.destacado = false; });
  guardarDatos();
}

function guardarDatos() {
  localStorage.setItem('champa_trabajadores', JSON.stringify(trabajadores));
  localStorage.setItem('champa_usuario', JSON.stringify(usuarioActual));
}

// ADMIN
function mostrarAdmin() {
  document.getElementById('app').classList.add('hidden');
  document.getElementById('admin').classList.remove('hidden');
  renderizarAdmin();
}

function renderizarAdmin() {
  document.getElementById('admin').innerHTML = `
    <div style="padding:20px;">
      <h2>Panel Admin ChampaPy</h2>
      <button class="btn" onclick="location.href=location.pathname">Volver a la app</button>
      <div id="lista-admin"></div>
    </div>
  `;
  document.getElementById('lista-admin').innerHTML = trabajadores.map(t => `
    <div class="card">
      <h4>${t.nombre} - ${t.oficio}</h4>
      <p>Tel: ${t.telefono} | Zona: ${t.zona}</p>
      <p>Créditos: ${t.creditos || 0} | Verificado: ${t.verificado? 'Sí' : 'No'}</p>
      <p>Destacado hasta: ${t.destacado_hasta? new Date(t.destacado_hasta).toLocaleDateString() : 'No'}</p>
      <button class="btn" onclick="sumarCredito('${t.id}', 1)">+1 Crédito</button>
      <button class="btn btn-destacado" onclick="sumarCredito('${t.id}', 3)">+3 Créditos</button>
      <button class="btn" onclick="verificar('${t.id}')">Verificar</button>
      <button class="btn btn-danger" onclick="borrar('${t.id}')">Borrar</button>
    </div>
  `).join('');
}

function sumarCredito(id, cant) {
  let t = trabajadores.find(x => x.id === id);
  t.creditos = (t.creditos || 0) + cant;
  guardarDatos(); renderizarAdmin();
  alert(`Agregados ${cant} créditos a ${t.nombre}`);
}

function verificar(id) {
  let t = trabajadores.find(x => x.id === id);
  t.verificado = true;
  guardarDatos(); renderizarAdmin();
}

function borrar(id) {
  if (confirm('Seguro?')) {
    trabajadores = trabajadores.filter(x => x.id!== id);
    guardarDatos(); renderizarAdmin();
  }
}
</script>
</body>
</html>
