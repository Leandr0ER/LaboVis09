<script>
  import * as d3 from "d3";

  export let lines = [];
  export let width = null; // 'width' se pasará desde el componente padre

  let svg; // Variable para enlazar al elemento <svg>

  // --- INICIO DE CAMBIOS DEL PASO 1.2 ---

  // 1. Definir las constantes de diseño
  const firstColumnWidth = 150; // Ancho de la primera columna (nombres de archivo)
  const fileInfoMargin = 100; // Margen entre la información del archivo y los puntos
  const dotsColumnX = firstColumnWidth + fileInfoMargin; // Posición X donde empiezan los puntos
  const approxDotWidth = 10; // Ancho aproximado de cada punto (para calcular cuántos caben por fila)
  const linesPerDot = 1; // Cuántas líneas de código representa un solo punto (aquí 1:1)
  const baseY = 10; // Posición Y base para el nombre del archivo
  const totalLinesOffset = 20; // Desplazamiento Y para el conteo de líneas (debajo del nombre)
  const fileInfoHeight = baseY + totalLinesOffset; // Altura total de la sección de información del archivo
  const dotRowHeight = 20; // Altura de cada fila de puntos

  // --- INICIO DE CAMBIO 1.6.1: Variable para recordar el conteo anterior de puntos ---
  let previousDotCounts = new Map();
  // --- FIN DE CAMBIO 1.6.1 ---

  // --- INICIO DE CAMBIO 1.3.1: Crear la escala de color ---
  // Esta escala asignará un color único a cada 'type' (lenguaje) de línea.
  //let colorScale = d3.scaleOrdinal(d3.schemeTableau10);

  // cambio 1.4
  export let colorScale = d3.scaleOrdinal(d3.schemeTableau10);
  // --- FIN DE CAMBIO 1.3.1 ---

  // --- INICIO DE CAMBIO 1.6.2: Modificación de generateDots para data-index ---
  function generateDots(file, svgWidth) {
    const totalDots = Math.ceil(file.lines.length / linesPerDot);
    const availableWidth = svgWidth - dotsColumnX;
    const maxDotsPerRow = Math.max(1, Math.floor(availableWidth / approxDotWidth));
    let tspans = "";
    const dotRows = Math.ceil(totalDots / maxDotsPerRow);

    // Lleva un contador de líneas dentro de la función para asignar data-index
    let lineIndexInFile = 0;

    for (let r = 0; r < dotRows; r++) {
      const rowLines = file.lines.slice(r * maxDotsPerRow, (r + 1) * maxDotsPerRow);
      
      const rowDots = rowLines
        .map(line => {
            const index = lineIndexInFile++; // Asigna el índice y lo incrementa
            return `<tspan class="dot" data-index="${index}" style="fill:${colorScale(line.type)};">•</tspan>`;
        })
        .join('');
      
      tspans += `<tspan x="${dotsColumnX}" dy="${r === 0 ? 0 : dotRowHeight + 'px'}">${rowDots}</tspan>`;
    }
    return tspans;
  }
  // --- FIN DE CAMBIO 1.6.2 ---

  let files = [];
  // --- INICIO DE CAMBIO 1.5: Ordenar los archivos ---
  $: files = d3.groups(lines, d => d.file)
              .map(([name, lines]) => ({ name, lines }))
              .sort((a, b) => b.lines.length - a.lines.length); // Añade esta línea
  // --- FIN DE CAMBIO 1.5 ---

  // 3. Calcular la altura requerida para cada archivo
  $: filesWithHeights = files.map(file => {
    const totalDots = Math.ceil(file.lines.length / linesPerDot);
    const availableWidth = width - dotsColumnX;
    const maxDotsPerRow = Math.max(1, Math.floor(availableWidth / approxDotWidth));
    const dotRows = Math.ceil(totalDots / maxDotsPerRow);
    return { ...file, groupHeight: fileInfoHeight + dotRows * dotRowHeight };
  });

  // 4. Calcular las posiciones Y acumuladas para cada grupo de archivo
  // Esto permite que cada archivo tenga una altura diferente
  $: positions = (() => {
    let pos = [], y = 0;
    for (const f of filesWithHeights) {
      pos.push(y);
      y += f.groupHeight;
    }
    return pos;
  })();

  // Bloque reactivo principal para dibujar y actualizar el SVG
  $: if (svg) {
    const svgWidth = width; // Usamos el 'width' pasado como prop
    // Calcula la altura total del SVG sumando la altura de todos los grupos
    const totalHeight = positions.length
        ? positions[positions.length - 1] + filesWithHeights[filesWithHeights.length - 1].groupHeight
        : 0;

    d3.select(svg)
      .attr('width', svgWidth)
      .attr('height', totalHeight)
      .style('overflow', 'hidden'); // Oculta cualquier contenido que se desborde

    const groups = d3.select(svg)
      .selectAll('g.file')
      .data(filesWithHeights, d => d.name); // Usamos filesWithHeights aquí

    // Eliminar grupos que ya no existen (para manejo de filtros)
    groups.exit().remove();

    // Añadir nuevos grupos (para nuevas entradas de datos)
    const enterGroups = groups.enter()
      .append('g')
      .attr('class', 'file')
      .attr('transform', (d, i) => `translate(0, ${positions[i]})`); // Usar las posiciones acumuladas

    // Añadir texto para el nombre del archivo en los nuevos grupos
    enterGroups.append('text')
      .attr('class', 'filename')
      .attr('x', 10)
      .attr('y', baseY) // Ajustar la posición Y inicial
      .attr('dominant-baseline', 'hanging')
      .text(d => d.name);

    // Añadir texto para el conteo de líneas (como un "small element" visualmente)
    enterGroups.append('text')
      .attr('class', 'linecount')
      .attr('x', 10)
      .attr('y', baseY + totalLinesOffset) // Debajo del nombre del archivo
      .attr('dominant-baseline', 'hanging')
      .text(d => `${d.lines.length} lines`);

    // 5. Añadir los puntos (dots) en los nuevos grupos
    enterGroups.append('text')
      .attr('class', 'unit-dots')
      .attr('x', dotsColumnX) // Empieza en la columna de puntos
      .attr('y', baseY - 2) // Ligeramente por encima de la línea base para los puntos
      .attr('dominant-baseline', 'mathematical') // mathematical es bueno para texto con símbolos como puntos
      .attr('fill', "#1f77b4") // Color por defecto para los puntos (será sobrescrito en el Paso 1.3)
      .html(d => generateDots(d, svgWidth)); // Generar el SVG de los puntos usando la función

    // --- INICIO DE CAMBIO 1.6.3: Lógica de animación para los grupos existentes ---
    // Actualización de los grupos existentes y la lógica de transición
    groups
        .attr('transform', (d, i) => `translate(0, ${positions[i]})`);

    groups.select('text.filename')
        .text(d => d.name);

    groups.select('text.linecount')
        .text(d => `${d.lines.length} lines`)
        .attr('x', 10);

    // Iterar sobre cada grupo para aplicar la lógica de transición
    groups.each(function(d) {
        const groupSel = d3.select(this);
        const unitDotsSel = groupSel.select('text.unit-dots');
        const newCount = d.lines.length;
        const oldCount = previousDotCounts.get(d.name) || 0; // Obtener el conteo anterior

        // Actualizar el HTML de los puntos ANTES de la transición
        unitDotsSel.html(generateDots(d, svgWidth));

        if (newCount > oldCount) {
            // Seleccionar solo los nuevos puntos (con data-index >= oldCount)
            unitDotsSel.selectAll('tspan.dot')
                .filter(function() {
                    return +this.getAttribute('data-index') >= oldCount;
                })
                .style('opacity', 0) // Empezar con opacidad 0
                .transition() // Iniciar la transición
                    .duration(1000) // Duración de 1 segundo
                    .ease(d3.easeCubicOut) // Función de easing
                    .style('opacity', 1); // Animar a opacidad 1
        }
        
        // Actualizar el conteo anterior para la próxima iteración
        previousDotCounts.set(d.name, newCount);
    });
    // --- FIN DE CAMBIO 1.6.3 ---
  }
  // --- FIN DE CAMBIOS DEL PASO 1.2 ---
</script>

<svg bind:this={svg}></svg>

<style>
  /* --- INICIO DE ESTILOS DEL PASO 1.2 --- */
  /* Usar :global() para asegurar que los estilos se apliquen a elementos creados dinámicamente por D3 */
  :global(g.file text.filename) {
    font-size: 14px;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    font-weight: bold;
    fill: light-dark(black, white); /* Estilo sugerido para tema claro/oscuro */
  }

  :global(g.file text.linecount) {
    font-size: 11px; /* Más pequeño para que parezca un elemento <small> */
    fill: grey; /* Color gris */
  }

  :global(g.file text.unit-dots) {
    font-size: 2.2rem; /* Tamaño grande para los puntos para que sean visibles */
    line-height: 1; /* Ajustar el interlineado para evitar superposiciones verticales */
  }
  /* --- FIN DE ESTILOS DEL PASO 1.2 --- */
</style>