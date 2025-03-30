<script lang="ts">
  import { page } from '$app/stores';
  import { goto } from '$app/navigation';
  
  let width = 12;
  let height = 12;
  let currentSeed = 111;
  
  // Get seed from URL or default to 0
  $: {
    const urlParam = $page.url.searchParams.get('seed');
    const urlSeed = urlParam ? parseInt(urlParam) : null;
    
    if (urlSeed === null || isNaN(urlSeed)) {
      currentSeed = 100010001000;
      goto(`?seed=100010001000`, { keepFocus: true });
    } else {
      currentSeed = urlSeed;
    }
  }
  
  const TERRAIN_TYPES = {
    MARCH: { id: 1, name: 'March', color: '#8B8589' },
    HEATH: { id: 2, name: 'Heath', color: '#B5998E' },
    CRAG: { id: 3, name: 'Crag', color: '#7D7769' },
    PEAKS: { id: 4, name: 'Peaks', color: '#726E6D' },
    FOREST: { id: 5, name: 'Forest', color: '#2D4F1E' },
    VALLEY: { id: 6, name: 'Valley', color: '#90A583' },
    HILLS: { id: 7, name: 'Hills', color: '#8B7355' },
    MEADOWS: { id: 8, name: 'Meadows', color: '#90B44B' },
    BOG: { id: 9, name: 'Bog', color: '#5B6356' },
    LAKE: { id: 10, name: 'Lake', color: '#58B2DC' },
    GLADE: { id: 11, name: 'Glade', color: '#A8D8B9' },
    PLAINS: { id: 12, name: 'Plains', color: '#C7B370' }
  };

  // Calculate hex dimensions
  const hexSize = 40; // Size of hexagon (radius)
  const hexWidth = hexSize * Math.sqrt(3);
  const hexHeight = hexSize * 2;
  const vertOffset = hexHeight * 0.75;
  
  // Better seeded random number generator
  function seededRandom(seed: number): number {
    // Using a more reliable seed-based random algorithm
    let t = seed += 0x6D2B79F5;
    t = Math.imul(t ^ t >>> 15, t | 1);
    t ^= t + Math.imul(t ^ t >>> 7, t | 61);
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  }
  
  interface Hex {
    x: number;
    y: number;
    barriers: boolean[];  // Array of 6 booleans, one for each side
  }

  interface TerrainType {
    id: number;
    name: string;
    color: string;
  }

  let terrainMap: TerrainType[][] = [];
  let hexGrid: Hex[] = [];

  function generateHexGrid(w: number, h: number): Hex[] {
    let hexes: Hex[] = [];
    for (let row = 0; row < h; row++) {
      for (let col = 0; col < w; col++) {
        const x = col * hexWidth + (row % 2) * (hexWidth / 2);
        const y = row * vertOffset;
        hexes.push({ x, y, barriers: [false, false, false, false, false, false] });
      }
    }
    return hexes;
  }
  
  // Generate hex path for a single hexagon with individual line barriers
  function getHexPath(x: number, y: number, barriers: boolean[], forFill: boolean = false): string {
    const commands = [];
    
    for (let i = 0; i < 6; i++) {
      const angle = (i * 60 - 30) * Math.PI / 180;
      const nextAngle = ((i + 1) * 60 - 30) * Math.PI / 180;
      const currentPoint = `${x + hexSize * Math.cos(angle)},${y + hexSize * Math.sin(angle)}`;
      const nextPoint = `${x + hexSize * Math.cos(nextAngle)},${y + hexSize * Math.sin(nextAngle)}`;
      
      if (forFill) {
        // For filled hex, create a complete path
        if (i === 0) {
          commands.push(`M${currentPoint}`);
        }
        commands.push(`L${nextPoint}`);
      } else if (barriers[i]) {
        // For barriers, only draw the barrier edges
        commands.push(`M${currentPoint}L${nextPoint}`);
      }
    }
    
    if (forFill) {
      commands.push('Z'); // Close the path for filling
    }
    
    return commands.join(' ');
  }

  // Generate paths for dotted borders
  function getDottedBorderPath(x: number, y: number): string {
    const points = [];
    const commands = [];
    
    for (let i = 0; i < 6; i++) {
      const angle = (i * 60 - 30) * Math.PI / 180;
      const nextAngle = ((i + 1) * 60 - 30) * Math.PI / 180;
      const currentPoint = `${x + hexSize * Math.cos(angle)},${y + hexSize * Math.sin(angle)}`;
      const nextPoint = `${x + hexSize * Math.cos(nextAngle)},${y + hexSize * Math.sin(nextAngle)}`;
      commands.push(`M${currentPoint}L${nextPoint}`);
    }
    
    return commands.join(' ');
  }

  function generateNewMap(seed: number = currentSeed): void {
    // Use the seed directly for each random call
    const random = () => seededRandom(seed++);
    
    // Initialize empty terrain map and terrain counters
    terrainMap = Array(height).fill(null).map(() => Array(width).fill(null));
    const terrainCounts = Object.fromEntries(
      Object.values(TERRAIN_TYPES).map(t => [t.id, 0])
    );

    // Generate hex grid with barriers
    hexGrid = generateHexGrid(width, height);
    const totalHexes = width * height;
    const targetBarriers = 24; // Fixed number of barriers instead of random range
    let barriersPlaced = 0;
    let attempts = 0;
    const maxAttempts = totalHexes * 2; // Prevent infinite loops
    
    // Create individual barriers
    while (barriersPlaced < targetBarriers && attempts < maxAttempts) {
      attempts++;
      // Pick a random hex
      const hexIdx = Math.floor(random() * totalHexes);
      const hex = hexGrid[hexIdx];
      const row = Math.floor(hexIdx / width);
      const col = hexIdx % width;
      
      // Only proceed if this hex has no barriers
      if (!hex.barriers.some(b => b)) {
        // Pick a random edge (0-5)
        const edge = Math.floor(random() * 6);
        const [nextX, nextY] = getAdjacentHexCoord(col, row, edge);
        
        // Check if adjacent hex exists and has no barriers
        if (isValidCoord(nextX, nextY)) {
          const adjacentHexIndex = nextY * width + nextX;
          const adjacentHex = hexGrid[adjacentHexIndex];
          
          // Only add barriers if both hexes are free of barriers
          if (adjacentHex && !adjacentHex.barriers.some(b => b)) {
            hex.barriers[edge] = true;
            adjacentHex.barriers[(edge + 3) % 6] = true;
            barriersPlaced++;
          }
        }
      }
    }
    
    // Helper function to get adjacent hex coordinates
    function getAdjacentHexCoord(x: number, y: number, direction: number): [number, number] {
      const even = y % 2 === 0;
      switch (direction) {
        case 0: return [even ? x - 1 : x, y - 1];     // top-left
        case 1: return [even ? x : x + 1, y - 1];     // top-right
        case 2: return [x + 1, y];                    // right
        case 3: return [even ? x : x + 1, y + 1];     // bottom-right
        case 4: return [even ? x - 1 : x, y + 1];     // bottom-left
        case 5: return [x - 1, y];                    // left
        default: return [x, y];
      }
    }
    
    // Generate clusters
    const numClusters = Math.floor((width * height) / 8); // More clusters, but they'll be smaller
    
    for (let i = 0; i < numClusters; i++) {
      // Filter available terrain types (those under 12 tiles)
      const availableTerrains = Object.values(TERRAIN_TYPES).filter(t => terrainCounts[t.id] < 12);
      if (availableTerrains.length === 0) break;
      
      // Weight terrain types by how many spaces they have left
      const weightedTerrains = availableTerrains.map(t => {
        const spacesLeft = 12 - terrainCounts[t.id];
        // Repeat each terrain type based on spaces left to weight the selection
        return Array(spacesLeft).fill(t);
      }).flat();
      
      const terrainType = weightedTerrains[Math.floor(random() * weightedTerrains.length)];
      const startX = Math.floor(random() * width);
      const startY = Math.floor(random() * height);
      // Smaller clusters - max size of 6 instead of 12
      const maxClusterSize = Math.min(6, 12 - terrainCounts[terrainType.id]);
      const clusterSize = Math.floor(random() * maxClusterSize) + 1;
      
      generateClusterSeeded(startX, startY, terrainType, clusterSize, random, terrainCounts);
    }
    
    // Fill remaining null hexes with random terrain
    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        if (terrainMap[y][x] === null) {
          // Filter available terrain types again
          const availableTerrains = Object.values(TERRAIN_TYPES).filter(t => terrainCounts[t.id] < 12);
          if (availableTerrains.length === 0) {
            // If all terrains are at max, use the ones with lowest counts
            const minCount = Math.min(...Object.values(terrainCounts));
            const resetTerrains = Object.values(TERRAIN_TYPES).filter(t => terrainCounts[t.id] === minCount);
            terrainMap[y][x] = resetTerrains[Math.floor(random() * resetTerrains.length)];
          } else {
            // Weight terrain types by how many spaces they have left
            const weightedTerrains = availableTerrains.map(t => {
              const spacesLeft = 12 - terrainCounts[t.id];
              return Array(spacesLeft).fill(t);
            }).flat();
            
            const terrainType = weightedTerrains[Math.floor(random() * weightedTerrains.length)];
            terrainMap[y][x] = terrainType;
            terrainCounts[terrainType.id]++;
          }
        }
      }
    }
  }

  function generateClusterSeeded(
    startX: number, 
    startY: number, 
    terrainType: TerrainType, 
    size: number, 
    random: () => number, 
    terrainCounts: Record<number, number>
  ): void {
    if (size <= 0 || !isValidCoord(startX, startY) || terrainMap[startY][startX] !== null) return;
    
    terrainMap[startY][startX] = terrainType;
    terrainCounts[terrainType.id]++;
    size--;
    
    const neighbors = getAdjacentHexes(startX, startY);
    // Use seeded shuffle
    for (let i = neighbors.length - 1; i > 0; i--) {
      const j = Math.floor(random() * (i + 1));
      [neighbors[i], neighbors[j]] = [neighbors[j], neighbors[i]];
    }
    
    for (const [nx, ny] of neighbors) {
      if (size > 0 && terrainCounts[terrainType.id] < 12) {
        generateClusterSeeded(nx, ny, terrainType, size, random, terrainCounts);
      }
    }
  }

  function isValidCoord(x: number, y: number): boolean {
    return x >= 0 && x < width && y >= 0 && y < height;
  }

  function getAdjacentHexes(x: number, y: number): [number, number][] {
    const neighbors: [number, number][] = [];
    const even = y % 2 === 0;
    const directions = [
      [even ? -1 : 0, -1], [even ? 0 : 1, -1],
      [-1, 0], [1, 0],
      [even ? -1 : 0, 1], [even ? 0 : 1, 1]
    ];
    
    for (const [dx, dy] of directions) {
      const newX = x + dx;
      const newY = y + dy;
      if (isValidCoord(newX, newY)) {
        neighbors.push([newX, newY]);
      }
    }
    return neighbors;
  }

  // Generate map when seed changes
  $: {
    generateNewMap(currentSeed);
  }
</script>

<div class="flex flex-col items-center p-4">
  <h1 class="text-2xl font-bold mb-4">Hex Map Generator</h1>
  
  <div class="mb-4 flex flex-col gap-4">
    <div class="flex items-center gap-2">
      <span>Width:</span>
      <input 
        type="number" 
        class="input input-bordered w-24" 
        bind:value={width}
        min="1"
        max="24"
      >
    </div>
    <div class="flex items-center gap-2">
      <span>Height:</span>
      <input 
        type="number" 
        class="input input-bordered w-24" 
        bind:value={height}
        min="1"
        max="24"
      >
    </div>
    <button class="btn btn-primary w-full" on:click={() => {
      const newSeed = Math.floor(Math.random() * 1000000000);
      goto(`?seed=${newSeed}`, { keepFocus: true });
    }}>
      Generate New Map
    </button>
    
    <div class="flex items-center gap-2 px-4">
      <span>Seed:</span>
      <input 
        type="number" 
        class="input input-bordered w-24" 
        value={currentSeed}
        on:input={(e) => {
          const target = e.target as HTMLInputElement;
          const newSeed = parseInt(target.value);
          goto(`?seed=${newSeed}`, { keepFocus: true });
        }}
      >
    </div>
  </div>

  <svg 
    width={width * hexWidth + hexWidth}
    height={height * vertOffset + hexHeight/2}
    class="border border-gray-300"
  >
    {#each hexGrid as hex, i}
      {@const col = i % width}
      {@const row = Math.floor(i / width)}
      <g>
        <!-- Filled hex -->
        <path 
          d={getHexPath(hex.x + hexSize, hex.y + hexSize, [], true)} 
          fill={terrainMap[row][col].color}
          stroke="none"
        />
        <!-- Dotted borders -->
        <path 
          d={getDottedBorderPath(hex.x + hexSize, hex.y + hexSize)} 
          fill="none"
          stroke="black" 
          stroke-width="2"
          stroke-dasharray="3,6"
          stroke-linecap="round"
        />
      </g>
    {/each}
    <!-- Draw all barriers in a separate layer on top -->
    {#each hexGrid as hex, i}
      {#if hex.barriers.some(b => b)}
        <path 
          d={getHexPath(hex.x + hexSize, hex.y + hexSize, hex.barriers)} 
          fill="none"
          stroke="red" 
          stroke-width="3"
          stroke-linecap="round"
        />
      {/if}
    {/each}
  </svg>

  <div class="grid grid-cols-4 gap-2 mt-4">
    {#each Object.values(TERRAIN_TYPES) as terrain}
      <div class="flex items-center">
        <div class="w-4 h-4 mr-2" style="background-color: {terrain.color}"></div>
        <span>{terrain.name}</span>
      </div>
    {/each}
  </div>
</div>
