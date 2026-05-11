
<script setup>
import { onMounted, onBeforeUnmount, ref } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { Chess } from 'chess.js'

const contenedor3D = ref(null)
const chess = new Chess() // Motor de ajedrez

let escena, camara, renderizador, controles
let raycaster, raton
let casillas = []
let piezas3D = new Map() // key: 'e2' → mesh
let piezaSeleccionada = null
let legalMovesHighlights = []

const tamañoCasilla = 4
const offset = 14 // para centrar el tablero

onMounted(() => {
  initEscena()
  crearTablero()
  crearTodasLasPiezas()
  setupEventos()
  animar()
})

function initEscena() {
  escena = new THREE.Scene()
  escena.background = new THREE.Color(0x0a0a12)
  escena.fog = new THREE.Fog(0x0a0a12, 60, 180)

  camara = new THREE.PerspectiveCamera(48, 900/650, 0.1, 200)
  camara.position.set(30, 45, 55)

  renderizador = new THREE.WebGLRenderer({ antialias: true })
  renderizador.setSize(900, 650)
  renderizador.shadowMap.enabled = true
  renderizador.shadowMap.type = THREE.PCFSoftShadowMap
  contenedor3D.value.appendChild(renderizador.domElement)

  // Iluminación
  escena.add(new THREE.AmbientLight(0xffffff, 0.7))
  const dirLight = new THREE.DirectionalLight(0xfff8e1, 1.3)
  dirLight.position.set(25, 60, 35)
  dirLight.castShadow = true
  escena.add(dirLight)
}

function crearTablero() {
  for (let fila = 0; fila < 8; fila++) {
    for (let col = 0; col < 8; col++) {
      const esClaro = (fila + col) % 2 === 0
      const material = new THREE.MeshStandardMaterial({
        color: esClaro ? 0xf0d9b5 : 0x5c4033,
        metalness: 0.05,
        roughness: 0.7
      })

      const casilla = new THREE.Mesh(
        new THREE.PlaneGeometry(tamañoCasilla, tamañoCasilla),
        material
      )
      casilla.rotation.x = -Math.PI / 2
      casilla.position.set(
        col * tamañoCasilla - offset,
        0.02,
        fila * tamañoCasilla - offset
      )
      casilla.userData = { fila, col, square: getSquareName(col, fila) }
      escena.add(casilla)
      casillas.push(casilla)
    }
  }
}

function getSquareName(col, fila) {
  const files = 'abcdefgh'
  return files[col] + (8 - fila)
}

// === PIEZAS PROCEDURALES CON EFECTO TORNASOL ===
function crearPieza(tipo, color) {
  const group = new THREE.Group()
  const esBlanca = color === 'w'

  const material = new THREE.MeshPhysicalMaterial({
    color: esBlanca ? 0xeeeeee : 0x222222,
    metalness: 0.85,
    roughness: 0.15,
    clearcoat: 1,
    clearcoatRoughness: 0.1,
    iridescence: 0.9,
    iridescenceIOR: 1.6,
    iridescenceThicknessRange: [80, 500],
    envMapIntensity: 1.3
  })

  let mesh

  switch (tipo) {
    case 'p': // Peón
      mesh = new THREE.Mesh(new THREE.CylinderGeometry(0.8, 1, 2.5, 32), material)
      break
    case 'r': // Torre
      mesh = new THREE.Mesh(new THREE.CylinderGeometry(1.1, 1.1, 4, 32), material)
      break
    case 'n': // Caballo
      mesh = new THREE.Mesh(new THREE.SphereGeometry(1.6, 32, 32), material)
      break
    case 'b': // Alfil
      mesh = new THREE.Mesh(new THREE.ConeGeometry(1.4, 4, 32), material)
      break
    case 'q': // Reina
      mesh = new THREE.Mesh(new THREE.CylinderGeometry(1.3, 0.8, 5, 32), material)
      break
    case 'k': // Rey
      mesh = new THREE.Mesh(new THREE.CylinderGeometry(1.4, 1, 5.5, 32), material)
      break
  }

  mesh.castShadow = true
  mesh.receiveShadow = true
  mesh.position.y = 2
  group.add(mesh)

  // Base
  const base = new THREE.Mesh(
    new THREE.CylinderGeometry(1.6, 1.8, 0.6, 32),
    material
  )
  base.position.y = 0.8
  group.add(base)

  group.userData = { tipo, color }
  group.scale.set(0.9, 0.9, 0.9)
  return group
}

function crearTodasLasPiezas() {
  const board = chess.board()

  for (let fila = 0; fila < 8; fila++) {
    for (let col = 0; col < 8; col++) {
      const pieza = board[fila][col]
      if (!pieza) continue

      const mesh = crearPieza(pieza.type, pieza.color)
      const x = col * tamañoCasilla - offset
      const z = fila * tamañoCasilla - offset

      mesh.position.set(x, 0, z)
      escena.add(mesh)

      const square = getSquareName(col, fila)
      piezas3D.set(square, mesh)
    }
  }
}

function setupEventos() {
  raycaster = new THREE.Raycaster()
  raton = new THREE.Vector2()

  renderizador.domElement.addEventListener('click', onClick)
  controles = new OrbitControls(camara, renderizador.domElement)
  controles.enableDamping = true
  controles.dampingFactor = 0.1
  controles.minDistance = 35
  controles.maxDistance = 120
}

function onClick(event) {
  const rect = contenedor3D.value.getBoundingClientRect()
  raton.x = ((event.clientX - rect.left) / rect.width) * 2 - 1
  raton.y = -((event.clientY - rect.top) / rect.height) * 2 + 1

  raycaster.setFromCamera(raton, camara)
  const intersects = raycaster.intersectObjects(casillas)

  if (intersects.length === 0) return

  const casilla = intersects[0].object
  const square = casilla.userData.square

  if (piezaSeleccionada) {
    intentarMover(piezaSeleccionada, square)
    piezaSeleccionada = null
    limpiarHighlights()
  } else {
    const pieza = piezas3D.get(square)
    if (pieza && pieza.userData.color === chess.turn()) {
      piezaSeleccionada = square
      mostrarMovimientosLegales(square)
    }
  }
}

function intentarMover(desde, hasta) {
  try {
    const movimiento = chess.move({ from: desde, to: hasta, promotion: 'q' })
    if (movimiento) {
      actualizarPosicion3D(desde, hasta)
    }
  } catch (e) {
    console.log("Movimiento ilegal")
  }
}

function actualizarPosicion3D(desde, hasta) {
  const mesh = piezas3D.get(desde)
  if (!mesh) return

  piezas3D.delete(desde)
  piezas3D.set(hasta, mesh)

  const col = 'abcdefgh'.indexOf(hasta[0])
  const fila = 8 - parseInt(hasta[1])

  mesh.position.set(
    col * tamañoCasilla - offset,
    mesh.position.y,
    fila * tamañoCasilla - offset
  )
}

function mostrarMovimientosLegales(square) {
  limpiarHighlights()
  const moves = chess.moves({ square, verbose: true })

  moves.forEach(move => {
    const casillaTarget = casillas.find(c => c.userData.square === move.to)
    if (casillaTarget) {
      casillaTarget.material.emissive = new THREE.Color(0x00ff88)
      casillaTarget.material.emissiveIntensity = 0.4
      legalMovesHighlights.push(casillaTarget)
    }
  })
}

function limpiarHighlights() {
  legalMovesHighlights.forEach(casilla => {
    casilla.material.emissive = new THREE.Color(0x000000)
    casilla.material.emissiveIntensity = 0
  })
  legalMovesHighlights = []
}

function animar() {
  requestAnimationFrame(animar)
  controles.update()
  renderizador.render(escena, camara)
}

onBeforeUnmount(() => {
  if (renderizador) renderizador.dispose()
})
</script>

<template>
  <div class="ajedrez-3d">
    <h1>♟️ Ajedrez 3D Tornasol - Jugable</h1>
    <div ref="contenedor3D" class="canvas"></div>
    <p>Click en una pieza → Click en casilla destino • Usa el ratón para rotar la vista</p>
  </div>
</template>

<style scoped>
.ajedrez-3d {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: radial-gradient(circle, #1a1a2e, #0a0a0a);
  min-height: 100vh;
  color: white;
  padding: 20px;
}

.canvas {
  width: 900px;
  height: 650px;
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 25px 70px rgba(156, 5, 5, 0.9);
  border: 2px solid #445577;
}
</style>
