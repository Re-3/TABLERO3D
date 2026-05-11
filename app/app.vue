<template>
  <div>
    <h1 style="text-align: center; color: white;">Chess Pro - Debug Mode</h1>
    <!-- ClientOnly es vital para que Three.js no choque con el servidor -->
    <ClientOnly>
      <TableroJuego />
    </ClientOnly>
  </div>
</template>

<style>ssssz
body { background: #1a1a1a; margin: 0; }
</style>
