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

  const SETTLEMENT_TYPES = {
    CASTLE: { name: 'Castle', symbol: '🏰' },
    TOWN: { name: 'Town', symbol: '🏘️' },
    FORTRESS: { name: 'Fortress', symbol: '🗼' },
    TOWER: { name: 'Tower', symbol: '🏛️' }
  };

  const LANDMARK_TYPES = {
    DWELLING: { name: 'Dwelling', symbol: '🏡', color: '#8B4513' },
    SANCTUM: { name: 'Sanctum', symbol: '⛪', color: '#FFD700' },
    MONUMENT: { name: 'Monument', symbol: '🗿', color: '#808080' },
    HAZARD: { name: 'Hazard', symbol: '⚠️', color: '#FF4500' },
    CURSE: { name: 'Curse', symbol: '👻', color: '#800080' },
    RUIN: { name: 'Ruin', symbol: '🏚️', color: '#696969' }
  };

  const MYTH_TYPES = [
    'The Apparatus', 'The Axe', 'The Bat', 'The Beast', 'The Blade', 'The Boar',
    'The Bull', 'The Catacomb', 'The Cave', 'The Changeling', 'The Chariot',
    'The Child', 'The Citadel', 'The Colossus', 'The Coven', 'The Crown',
    'The Cudgel', 'The Dead', 'The Demon', 'The Desert', 'The Dwarf', 'The Eagle',
    'The Elephant', 'The Elf', 'The Eye', 'The Forest', 'The Fortress', 'The Gargoyle',
    'The Glade', 'The Goblin', 'The Harp', 'The Hole', 'The Hound', 'The Hydra',
    'The Imp', 'The Inferno', 'The Judge', 'The Legion', 'The Lich', 'The Lion',
    'The Lizard', 'The Mist', 'The Moon', 'The Mountain', 'The Ogre', 'The Order',
    'The Pack', 'The Plague', 'The Pool', 'The River', 'The Rock', 'The Sea',
    'The Shadow', 'The Snail', 'The Spectre', 'The Spider', 'The Spire', 'The Sprite',
    'The Star', 'The Sun', 'The Toad', 'The Tournament', 'The Tower', 'The Tree',
    'The Troll', 'The Underworld', 'The Wall', 'The Wheel', 'The Wight', 'The Wraith',
    'The Wurm', 'The Wyvern'
  ];

  interface Settlement {
    type: keyof typeof SETTLEMENT_TYPES;
    x: number;
    y: number;
    seatOfPower?: boolean;
  }

  interface Myth {
    name: string;
    x: number;
    y: number;
  }

  interface Landmark {
    type: keyof typeof LANDMARK_TYPES;
    x: number;
    y: number;
  }

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
    hasRiver?: boolean;   // Whether this hex contains a river
    riverFrom?: number;   // Direction river comes from (0-5)
    riverTo?: number;     // Direction river goes to (0-5)
  }

  interface TerrainType {
    id: number;
    name: string;
    color: string;
  }

  let terrainMap: TerrainType[][] = [];
  let hexGrid: Hex[] = [];
  let settlements: Settlement[] = [];
  let myths: Myth[] = [];
  let landmarks: Landmark[] = [];

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

  function findAllLakes(): { size: number, startX: number, startY: number }[] {
    let lakes: { size: number, startX: number, startY: number }[] = [];
    let visited = Array(height).fill(null).map(() => Array(width).fill(false));

    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        if (!visited[y][x] && terrainMap[y][x].name === 'Lake') {
          let size = 0;
          let stack = [[x, y]];
          while (stack.length > 0) {
            const [cx, cy] = stack.pop()!;
            if (visited[cy][cx]) continue;
            visited[cy][cx] = true;
            size++;
            
            for (const [nx, ny] of getAdjacentHexes(cx, cy)) {
              if (!visited[ny][nx] && terrainMap[ny][nx].name === 'Lake') {
                stack.push([nx, ny]);
              }
            }
          }
          if (size >= 3) { // Only track lakes of size 3 or more
            lakes.push({ size, startX: x, startY: y });
          }
        }
      }
    }
    return lakes;
  }

  function generateRiver(startX: number, startY: number, random: () => number): void {
    let currentX = startX;
    let currentY = startY;
    let visited = new Set<string>();

    // Start hex is part of a lake, so we need to find a non-lake hex to start from
    const startHexIndex = startY * width + startX;
    const startHex = hexGrid[startHexIndex];
    startHex.hasRiver = true;
    
    // Find a non-lake neighbor to start the river from
    const startNeighbors = getAdjacentHexes(startX, startY)
      .filter(([nx, ny]) => terrainMap[ny][nx].name !== 'Lake');
    
    if (startNeighbors.length === 0) return; // No valid starting point
    
    // Choose the neighbor that's most towards the target edge
    const targetX = startX < width/2 ? width - 1 : 0;
    const targetY = startY < height/2 ? height - 1 : 0;
    
    startNeighbors.sort(([ax, ay], [bx, by]) => {
      const aScore = Math.abs(ax - targetX) + Math.abs(ay - targetY);
      const bScore = Math.abs(bx - targetX) + Math.abs(by - targetY);
      return aScore - bScore;
    });
    
    const [nextX, nextY] = startNeighbors[0];
    const direction = getDirectionBetweenHexes(startX, startY, nextX, nextY);
    startHex.riverTo = direction;
    
    const nextHex = hexGrid[nextY * width + nextX];
    nextHex.hasRiver = true;
    nextHex.riverFrom = (direction + 3) % 6;
    
    currentX = nextX;
    currentY = nextY;
    visited.add(`${startX},${startY}`);
    visited.add(`${currentX},${currentY}`);
    
    while (true) {
      const hex = hexGrid[currentY * width + currentX];
      
      // Get valid neighbors
      const neighbors = getAdjacentHexes(currentX, currentY)
        .filter(([nx, ny]) => {
          const key = `${nx},${ny}`;
          if (visited.has(key)) return false;
          // Allow moving into another lake for termination
          if (terrainMap[ny][nx].name === 'Lake') return true;
          return true;
        });
      
      if (neighbors.length === 0) break;
      
      // Check if any neighbor is a lake (prioritize connecting to lakes)
      const lakePath = neighbors.find(([nx, ny]) => terrainMap[ny][nx].name === 'Lake');
      if (lakePath) {
        // Connect to the lake and end the river
        const [nextX, nextY] = lakePath;
        const direction = getDirectionBetweenHexes(currentX, currentY, nextX, nextY);
        hex.riverTo = direction;
        const nextHex = hexGrid[nextY * width + nextX];
        nextHex.hasRiver = true;
        nextHex.riverFrom = (direction + 3) % 6;
        break;
      }
      
      // Score neighbors based on distance to target
      const scoredNeighbors = neighbors.map(([nx, ny]) => {
        const distanceToTarget = Math.abs(nx - targetX) + Math.abs(ny - targetY);
        const randomFactor = random() * 2; // Add some randomness to prevent straight lines
        return { pos: [nx, ny], score: distanceToTarget + randomFactor };
      });
      
      // Sort by score (lower is better)
      scoredNeighbors.sort((a, b) => a.score - b.score);
      
      // Pick from the best options with some randomness
      const bestOptions = scoredNeighbors.slice(0, Math.min(2, scoredNeighbors.length));
      const [nextX, nextY] = bestOptions[Math.floor(random() * bestOptions.length)].pos;
      
      // Find the direction between hexes
      const direction = getDirectionBetweenHexes(currentX, currentY, nextX, nextY);
      hex.riverTo = direction;
      
      // Set the river direction for the next hex
      const nextHex = hexGrid[nextY * width + nextX];
      nextHex.hasRiver = true;
      nextHex.riverFrom = (direction + 3) % 6;
      
      // If we've reached an edge, end the river
      if (nextX === 0 || nextX === width - 1 || nextY === 0 || nextY === height - 1) {
        break;
      }
      
      visited.add(`${nextX},${nextY}`);
      currentX = nextX;
      currentY = nextY;
    }
  }

  function getDirectionBetweenHexes(x1: number, y1: number, x2: number, y2: number): number {
    const even1 = y1 % 2 === 0;
    if (y2 < y1) return x2 >= (even1 ? x1 : x1 + 1) ? 1 : 0;
    if (y2 > y1) return x2 >= (even1 ? x1 : x1 + 1) ? 3 : 4;
    return x2 > x1 ? 2 : 5;
  }

  function ensureLargeLake(random: () => number): void {
    const lakes = findAllLakes();
    const hasLargeLake = lakes.some(lake => lake.size >= 4);
    if (hasLargeLake) return;  // Already have a large enough lake
    
    // Create a new lake cluster
    const lakeType = TERRAIN_TYPES.LAKE;
    const startX = Math.floor(random() * width);
    const startY = Math.floor(random() * height);
    const targetSize = 4 + Math.floor(random() * 3);  // Size 4-6
    
    let stack = [[startX, startY]];
    let placed = 0;
    
    while (stack.length > 0 && placed < targetSize) {
      const [x, y] = stack.pop()!;
      if (!isValidCoord(x, y) || terrainMap[y][x].name === 'Lake') continue;
      
      terrainMap[y][x] = lakeType;
      placed++;
      
      // Add neighbors in random order
      const neighbors = getAdjacentHexes(x, y);
      for (let i = neighbors.length - 1; i > 0; i--) {
        const j = Math.floor(random() * (i + 1));
        [neighbors[i], neighbors[j]] = [neighbors[j], neighbors[i]];
      }
      stack.push(...neighbors);
    }
  }

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

  function generateMyths(random: () => number, existingSettlements: Settlement[]): Myth[] {
    const newMyths: Myth[] = [];
    const minDistanceFromSettlements = Math.floor(Math.min(width, height) / 3);
    const minDistanceBetweenMyths = Math.floor(Math.min(width, height) / 4);
    
    function getDistance(x1: number, y1: number, x2: number, y2: number): number {
      return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
    }
    
    function isValidPosition(x: number, y: number): boolean {
      // Check distance from settlements
      for (const settlement of existingSettlements) {
        if (getDistance(x, y, settlement.x, settlement.y) < minDistanceFromSettlements) {
          return false;
        }
      }
      
      // Check distance from other myths
      for (const myth of newMyths) {
        if (getDistance(x, y, myth.x, myth.y) < minDistanceBetweenMyths) {
          return false;
        }
      }
      
      // Check if the terrain is suitable (not a lake)
      return terrainMap[y][x].name !== 'Lake';
    }
    
    // Create a shuffled copy of myth types
    const availableMyths = [...MYTH_TYPES];
    for (let i = availableMyths.length - 1; i > 0; i--) {
      const j = Math.floor(random() * (i + 1));
      [availableMyths[i], availableMyths[j]] = [availableMyths[j], availableMyths[i]];
    }
    
    let attempts = 0;
    const maxAttempts = 200;
    
    while (newMyths.length < 6 && attempts < maxAttempts) {
      attempts++;
      
      const x = Math.floor(random() * width);
      const y = Math.floor(random() * height);
      
      if (isValidPosition(x, y)) {
        const mythName = availableMyths[newMyths.length];
        newMyths.push({ name: mythName, x, y });
      }
    }
    
    return newMyths;
  }

  function generateLandmarks(random: () => number, existingSettlements: Settlement[], existingMyths: Myth[]): Landmark[] {
    const newLandmarks: Landmark[] = [];
    const minDistanceFromSettlements = Math.floor(Math.min(width, height) / 6); // Reduced from 1/4 to 1/6
    const minDistanceFromMyths = Math.floor(Math.min(width, height) / 6); // Reduced from 1/4 to 1/6
    
    function getDistance(x1: number, y1: number, x2: number, y2: number): number {
      return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
    }
    
    function isValidPosition(x: number, y: number): boolean {
      // Check distance from settlements
      for (const settlement of existingSettlements) {
        if (getDistance(x, y, settlement.x, settlement.y) < minDistanceFromSettlements) {
          return false;
        }
      }
      
      // Check distance from myths
      for (const myth of existingMyths) {
        if (getDistance(x, y, myth.x, myth.y) < minDistanceFromMyths) {
          return false;
        }
      }
      
      // Check if the hex is already occupied by another landmark
      for (const landmark of newLandmarks) {
        if (landmark.x === x && landmark.y === y) {
          return false;
        }
      }
      
      // Check if the terrain is suitable (not a lake)
      return terrainMap[y][x].name !== 'Lake';
    }
    
    // Generate 3-4 of each landmark type
    const landmarkTypes = Object.keys(LANDMARK_TYPES) as (keyof typeof LANDMARK_TYPES)[];
    
    for (const type of landmarkTypes) {
      const count = 3 + Math.floor(random() * 2); // 3 or 4
      let attempts = 0;
      const maxAttempts = 100;
      let placed = 0;
      
      while (placed < count && attempts < maxAttempts) {
        attempts++;
        const x = Math.floor(random() * width);
        const y = Math.floor(random() * height);
        
        if (isValidPosition(x, y)) {
          newLandmarks.push({ type, x, y });
          placed++;
        }
      }
    }
    
    return newLandmarks;
  }

  function generateNewMap(seed: number = currentSeed): void {
    // Use the seed directly for each random call
    const random = () => seededRandom(seed++);
    
    // Initialize empty terrain map and terrain counters
    terrainMap = Array(height).fill(null).map(() => Array(width).fill(null));
    const terrainCounts = Object.fromEntries(
      Object.values(TERRAIN_TYPES)
        .filter(t => t.name !== 'Lake') // Exclude Lake from initial generation
        .map(t => [t.id, 0])
    );

    // Generate hex grid with barriers
    hexGrid = generateHexGrid(width, height);
    const totalHexes = width * height;
    const targetBarriers = 24;
    let barriersPlaced = 0;
    let attempts = 0;
    const maxAttempts = totalHexes * 2;
    
    // Create individual barriers
    while (barriersPlaced < targetBarriers && attempts < maxAttempts) {
      attempts++;
      const hexIdx = Math.floor(random() * totalHexes);
      const hex = hexGrid[hexIdx];
      const row = Math.floor(hexIdx / width);
      const col = hexIdx % width;
      
      if (!hex.barriers.some(b => b)) {
        const edge = Math.floor(random() * 6);
        const [nextX, nextY] = getAdjacentHexCoord(col, row, edge);
        
        if (isValidCoord(nextX, nextY)) {
          const adjacentHexIndex = nextY * width + nextX;
          const adjacentHex = hexGrid[adjacentHexIndex];
          
          if (adjacentHex && !adjacentHex.barriers.some(b => b)) {
            hex.barriers[edge] = true;
            adjacentHex.barriers[(edge + 3) % 6] = true;
            barriersPlaced++;
          }
        }
      }
    }
    
    // Generate terrain clusters (excluding lakes)
    const numClusters = Math.floor((width * height) / 8);
    
    for (let i = 0; i < numClusters; i++) {
      const availableTerrains = Object.values(TERRAIN_TYPES)
        .filter(t => t.name !== 'Lake' && terrainCounts[t.id] < 12);
      if (availableTerrains.length === 0) break;
      
      const weightedTerrains = availableTerrains.map(t => {
        const spacesLeft = 12 - terrainCounts[t.id];
        return Array(spacesLeft).fill(t);
      }).flat();
      
      const terrainType = weightedTerrains[Math.floor(random() * weightedTerrains.length)];
      const startX = Math.floor(random() * width);
      const startY = Math.floor(random() * height);
      const maxClusterSize = Math.min(6, 12 - terrainCounts[terrainType.id]);
      const clusterSize = Math.floor(random() * maxClusterSize) + 1;
      
      generateClusterSeeded(startX, startY, terrainType, clusterSize, random, terrainCounts);
    }
    
    // Fill remaining null hexes with random terrain (excluding lakes)
    for (let y = 0; y < height; y++) {
      for (let x = 0; x < width; x++) {
        if (terrainMap[y][x] === null) {
          const availableTerrains = Object.values(TERRAIN_TYPES)
            .filter(t => t.name !== 'Lake' && terrainCounts[t.id] < 12);
          
          if (availableTerrains.length === 0) {
            const minCount = Math.min(...Object.values(terrainCounts));
            const resetTerrains = Object.values(TERRAIN_TYPES)
              .filter(t => t.name !== 'Lake' && terrainCounts[t.id] === minCount);
            terrainMap[y][x] = resetTerrains[Math.floor(random() * resetTerrains.length)];
          } else {
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
    
    // Generate lakes after initial terrain
    generateLakes(random);
    
    // Generate one main river that connects opposite edges through a large lake
    generateMainRiver(random);
    
    // Generate settlements after everything else
    settlements = generateSettlements(random);
    
    // Generate myths after settlements
    myths = generateMyths(random, settlements);

    // Generate landmarks after myths
    landmarks = generateLandmarks(random, settlements, myths);
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

  // Function to get the point at a certain angle from hex center
  function getPointAtAngle(x: number, y: number, angle: number, distance: number = hexSize * 0.5): string {
    const radians = (angle - 30) * Math.PI / 180;
    return `${x + distance * Math.cos(radians)},${y + distance * Math.sin(radians)}`;
  }

  function getRiverPoints(): string {
    let riverPaths: string[] = [];
    let processedHexes = new Set<string>();
    
    // Process each hex
    for (let i = 0; i < hexGrid.length; i++) {
      const hex = hexGrid[i];
      const row = Math.floor(i / width);
      const col = i % width;
      const hexKey = `${col},${row}`;
      
      // Look for hexes that have rivers but no incoming river (starting points)
      // Also ensure we haven't processed this hex as part of another path
      if (hex.hasRiver && hex.riverFrom === undefined && !processedHexes.has(hexKey)) {
        const points: { x: number, y: number }[] = [];
        let currentHex = hex;
        let currentIndex = i;
        let isEdgeStart = col === 0 || col === width - 1 || row === 0 || row === height - 1;
        
        // If starting from edge, add an extended point beyond the map
        if (isEdgeStart) {
          const centerX = currentHex.x + hexSize;
          const centerY = currentHex.y + hexSize;
          const extensionDistance = hexSize * 1.5; // Increased from 0.5 to 1.5
          
          // Determine direction to extend based on which edge we're on
          let extendedX = centerX;
          let extendedY = centerY;
          
          if (col === 0) extendedX -= extensionDistance;
          if (col === width - 1) extendedX += extensionDistance;
          if (row === 0) extendedY -= extensionDistance;
          if (row === height - 1) extendedY += extensionDistance;
          
          points.push({ x: extendedX, y: extendedY });
        }
        
        // Build array of points for this river path
        while (currentHex && currentHex.hasRiver) {
          const row = Math.floor(currentIndex / width);
          const col = currentIndex % width;
          const currentKey = `${col},${row}`;
          
          // Add current point
          const centerX = currentHex.x + hexSize;
          const centerY = currentHex.y + hexSize;
          points.push({ x: centerX, y: centerY });
          processedHexes.add(currentKey);
          
          // Check for branches at this point
          for (let dir = 0; dir < 6; dir++) {
            if (dir === currentHex.riverTo) continue; // Skip main flow direction
            
            const [nextX, nextY] = getAdjacentHexCoord(col, row, dir);
            if (!isValidCoord(nextX, nextY)) continue;
            
            const nextIndex = nextY * width + nextX;
            const nextHex = hexGrid[nextIndex];
            const nextKey = `${nextX},${nextY}`;
            
            // If this neighbor has a river and hasn't been processed, it's a branch
            if (nextHex.hasRiver && !processedHexes.has(nextKey)) {
              // Create a new path for this branch
              const branchPoints: { x: number, y: number }[] = [];
              let branchHex = currentHex;
              let branchIndex = currentIndex;
              
              // Add the branch starting point
              branchPoints.push({ x: centerX, y: centerY });
              
              // Follow the branch
              let branchCurrent = nextHex;
              let branchCurrentIndex = nextIndex;
              processedHexes.add(nextKey);
              
              while (branchCurrent && branchCurrent.hasRiver) {
                const branchX = branchCurrent.x + hexSize;
                const branchY = branchCurrent.y + hexSize;
                branchPoints.push({ x: branchX, y: branchY });
                
                if (branchCurrent.riverTo !== undefined) {
                  const [nextBranchX, nextBranchY] = getAdjacentHexCoord(
                    Math.floor(branchCurrentIndex % width),
                    Math.floor(branchCurrentIndex / width),
                    branchCurrent.riverTo
                  );
                  if (isValidCoord(nextBranchX, nextBranchY)) {
                    const nextBranchIndex = nextBranchY * width + nextBranchX;
                    const nextBranchKey = `${nextBranchX},${nextBranchY}`;
                    if (!processedHexes.has(nextBranchKey)) {
                      processedHexes.add(nextBranchKey);
                      branchCurrentIndex = nextBranchIndex;
                      branchCurrent = hexGrid[nextBranchIndex];
                    } else {
                      break;
                    }
                  } else {
                    // If branch ends at edge, extend it
                    const lastX = branchCurrent.x + hexSize;
                    const lastY = branchCurrent.y + hexSize;
                    const extensionDistance = hexSize * 1.5; // Increased from 0.5 to 1.5
                    
                    // Determine direction to extend
                    let extendedX = lastX;
                    let extendedY = lastY;
                    const branchCol = Math.floor(branchCurrentIndex % width);
                    const branchRow = Math.floor(branchCurrentIndex / width);
                    
                    if (nextBranchX < 0) extendedX -= extensionDistance;
                    if (nextBranchX >= width) extendedX += extensionDistance;
                    if (nextBranchY < 0) extendedY -= extensionDistance;
                    if (nextBranchY >= height) extendedY += extensionDistance;
                    
                    branchPoints.push({ x: extendedX, y: extendedY });
                    break;
                  }
                } else {
                  break;
                }
              }
              
              // Generate SVG path for this branch
              if (branchPoints.length >= 2) {
                let branchPath = `M${branchPoints[0].x},${branchPoints[0].y}`;
                const tension = 0.3;
                
                for (let i = 0; i < branchPoints.length - 1; i++) {
                  const p0 = i > 0 ? branchPoints[i - 1] : branchPoints[i];
                  const p1 = branchPoints[i];
                  const p2 = branchPoints[i + 1];
                  const p3 = i < branchPoints.length - 2 ? branchPoints[i + 2] : p2;
                  
                  const controlPoint1 = {
                    x: p1.x + (p2.x - p0.x) * tension,
                    y: p1.y + (p2.y - p0.y) * tension
                  };
                  const controlPoint2 = {
                    x: p2.x - (p3.x - p1.x) * tension,
                    y: p2.y - (p3.y - p1.y) * tension
                  };
                  
                  branchPath += ` C${controlPoint1.x},${controlPoint1.y} ${controlPoint2.x},${controlPoint2.y} ${p2.x},${p2.y}`;
                }
                
                riverPaths.push(branchPath);
              }
            }
          }
          
          // Continue with main path
          if (currentHex.riverTo !== undefined) {
            const [nextX, nextY] = getAdjacentHexCoord(col, row, currentHex.riverTo);
            if (isValidCoord(nextX, nextY)) {
              currentIndex = nextY * width + nextX;
              currentHex = hexGrid[currentIndex];
            } else {
              // If main path ends at edge, extend it
              const lastX = currentHex.x + hexSize;
              const lastY = currentHex.y + hexSize;
              const extensionDistance = hexSize * 1.5; // Increased from 0.5 to 1.5
              
              // Determine direction to extend
              let extendedX = lastX;
              let extendedY = lastY;
              
              if (nextX < 0) extendedX -= extensionDistance;
              if (nextX >= width) extendedX += extensionDistance;
              if (nextY < 0) extendedY -= extensionDistance;
              if (nextY >= height) extendedY += extensionDistance;
              
              points.push({ x: extendedX, y: extendedY });
              break;
            }
          } else {
            break;
          }
        }
        
        // Generate SVG path with curves for the main river
        if (points.length >= 2) {
          let path = `M${points[0].x},${points[0].y}`;
          const tension = 0.3;
          
          for (let i = 0; i < points.length - 1; i++) {
            const p0 = i > 0 ? points[i - 1] : points[i];
            const p1 = points[i];
            const p2 = points[i + 1];
            const p3 = i < points.length - 2 ? points[i + 2] : p2;
            
            const controlPoint1 = {
              x: p1.x + (p2.x - p0.x) * tension,
              y: p1.y + (p2.y - p0.y) * tension
            };
            const controlPoint2 = {
              x: p2.x - (p3.x - p1.x) * tension,
              y: p2.y - (p3.y - p1.y) * tension
            };
            
            path += ` C${controlPoint1.x},${controlPoint1.y} ${controlPoint2.x},${controlPoint2.y} ${p2.x},${p2.y}`;
          }
          
          riverPaths.push(path);
        }
      }
    }
    
    return riverPaths.join(' ');
  }

  function getPointCoords(x: number, y: number, angle: number, distance: number): [number, number] {
    const radians = (angle - 30) * Math.PI / 180;
    return [
      x + distance * Math.cos(radians),
      y + distance * Math.sin(radians)
    ];
  }

  function generateLakes(random: () => number): void {
    const lakeType = TERRAIN_TYPES.LAKE;
    
    // Generate 1-3 large lakes (3-5 tiles)
    const numLargeLakes = 1 + Math.floor(random() * 3); // 1 to 3 large lakes
    for (let i = 0; i < numLargeLakes; i++) {
      const startX = Math.floor(random() * width);
      const startY = Math.floor(random() * height);
      const targetSize = 3 + Math.floor(random() * 3); // 3 to 5 tiles
      
      let stack = [[startX, startY]];
      let placed = 0;
      let visited = new Set<string>();
      
      while (stack.length > 0 && placed < targetSize) {
        const [x, y] = stack.pop()!;
        const key = `${x},${y}`;
        if (!isValidCoord(x, y) || visited.has(key) || terrainMap[y][x].name === 'Lake') continue;
        visited.add(key);
        
        terrainMap[y][x] = lakeType;
        placed++;
        
        // Add neighbors in random order
        const neighbors = getAdjacentHexes(x, y);
        for (let i = neighbors.length - 1; i > 0; i--) {
          const j = Math.floor(random() * (i + 1));
          [neighbors[i], neighbors[j]] = [neighbors[j], neighbors[i]];
        }
        stack.push(...neighbors);
      }
    }
    
    // Increased chance (40%) to add 1-2 small lakes (1-2 tiles)
    if (random() < 0.4) {
      const numSmallLakes = 1 + Math.floor(random() * 2);
      for (let i = 0; i < numSmallLakes; i++) {
        const startX = Math.floor(random() * width);
        const startY = Math.floor(random() * height);
        const targetSize = 1 + Math.floor(random() * 2);
        
        let placed = 0;
        let stack = [[startX, startY]];
        let visited = new Set<string>();
        
        while (stack.length > 0 && placed < targetSize) {
          const [x, y] = stack.pop()!;
          const key = `${x},${y}`;
          if (!isValidCoord(x, y) || visited.has(key) || terrainMap[y][x].name === 'Lake') continue;
          visited.add(key);
          
          terrainMap[y][x] = lakeType;
          placed++;
          
          if (placed < targetSize) {
            const neighbors = getAdjacentHexes(x, y);
            for (let i = neighbors.length - 1; i > 0; i--) {
              const j = Math.floor(random() * (i + 1));
              [neighbors[i], neighbors[j]] = [neighbors[j], neighbors[i]];
            }
            stack.push(...neighbors);
          }
        }
      }
    }
  }

  function generateMainRiver(random: () => number): void {
    // Find all large lakes (3+ tiles)
    const lakes = findAllLakes().filter(lake => lake.size >= 3);
    console.log('Found large lakes:', lakes);
    if (lakes.length === 0) {
      console.log('No large lakes found, skipping river generation');
      return;
    }
    
    // Choose a random large lake
    const targetLake = lakes[Math.floor(random() * lakes.length)];
    console.log('Selected target lake:', targetLake);
    
    // Clear any existing river data
    for (const hex of hexGrid) {
      hex.hasRiver = false;
      hex.riverFrom = undefined;
      hex.riverTo = undefined;
    }
    
    // Pick a random edge that's opposite to the lake's position
    let startX, startY;
    const lakeX = targetLake.startX;
    const lakeY = targetLake.startY;
    
    // Determine if we should do horizontal or vertical river
    if (Math.abs(lakeX - width/2) > Math.abs(lakeY - height/2)) {
      // Lake is more to the sides, do horizontal river
      startX = lakeX < width/2 ? width - 1 : 0;
      startY = Math.floor(random() * height);
      while (terrainMap[startY][startX].name === 'Lake') {
        startY = Math.floor(random() * height);
      }
      console.log('Generating horizontal river from:', { startX, startY });
    } else {
      // Lake is more to top/bottom, do vertical river
      startX = Math.floor(random() * width);
      startY = lakeY < height/2 ? height - 1 : 0;
      while (terrainMap[startY][startX].name === 'Lake') {
        startX = Math.floor(random() * width);
      }
      console.log('Generating vertical river from:', { startX, startY });
    }
    
    // Generate river from edge to lake
    generateRiverSegment(startX, startY, targetLake.startX, targetLake.startY, random);
  }

  function generateRiverSegment(startX: number, startY: number, targetX: number, targetY: number, random: () => number): void {
    console.log('Starting river segment from', { startX, startY }, 'to', { targetX, targetY });
    let currentX = startX;
    let currentY = startY;
    let visited = new Set<string>();
    let branchesCreated = 0;
    const maxBranches = 2; // Limit total number of branches
    
    // Set up the starting hex
    const startHex = hexGrid[startY * width + startX];
    startHex.hasRiver = true;
    visited.add(`${startX},${startY}`);
    
    // Determine initial direction based on starting edge
    const isStartLeft = startX === 0;
    const isStartRight = startX === width - 1;
    const isStartTop = startY === 0;
    const isStartBottom = startY === height - 1;
    
    let steps = 0;
    const maxSteps = width * height; // Prevent infinite loops
    
    // Function to find nearby lakes within 3 hexes
    function findNearbyLakes(x: number, y: number): [number, number][] {
      let nearbyLakes: [number, number][] = [];
      let checked = new Set<string>();
      let queue: [number, number, number][] = [[x, y, 0]]; // [x, y, distance]
      
      while (queue.length > 0) {
        const [cx, cy, dist] = queue.shift()!;
        const key = `${cx},${cy}`;
        if (checked.has(key)) continue;
        checked.add(key);
        
        // If we found a lake that's not our target and not already connected to a river
        if (dist > 0 && dist <= 3 && terrainMap[cy][cx].name === 'Lake' && 
            !(cx === targetX && cy === targetY) && 
            !hexGrid[cy * width + cx].hasRiver) {
          nearbyLakes.push([cx, cy]);
        }
        
        // If we haven't reached max distance, add neighbors
        if (dist < 3) {
          for (const [nx, ny] of getAdjacentHexes(cx, cy)) {
            if (!checked.has(`${nx},${ny}`)) {
              queue.push([nx, ny, dist + 1]);
            }
          }
        }
      }
      return nearbyLakes;
    }
    
    while (steps < maxSteps) {
      steps++;
      const hex = hexGrid[currentY * width + currentX];
      
      // Check if we've reached the target lake
      if (terrainMap[currentY][currentX].name === 'Lake' && 
          currentX === targetX && currentY === targetY) {
        console.log('River reached target lake at', { currentX, currentY });
        break;
      }
      
      // Only check for nearby lakes if we haven't created too many branches
      // and we're at least 3 steps into the river
      if (branchesCreated < maxBranches && steps > 3) {
        const nearbyLakes = findNearbyLakes(currentX, currentY);
        if (nearbyLakes.length > 0) {
          console.log('Found nearby lakes:', nearbyLakes);
          // Only create one branch at this point, to the closest lake
          const [lakeX, lakeY] = nearbyLakes[0];
          
          // Create a new branch from current position to the lake
          let branchX = currentX;
          let branchY = currentY;
          let branchVisited = new Set<string>(visited);
          let branchSteps = 0;
          const maxBranchSteps = 6; // Limit branch length
          
          while (branchSteps < maxBranchSteps) {
            branchSteps++;
            const branchHex = hexGrid[branchY * width + branchX];
            
            // Check if we've reached the lake
            if (branchX === lakeX && branchY === lakeY) {
              console.log('Branch reached lake at', { branchX, branchY });
              branchesCreated++;
              break;
            }
            
            // Get valid neighbors for the branch
            const branchNeighbors = getAdjacentHexes(branchX, branchY)
              .filter(([nx, ny]) => {
                const key = `${nx},${ny}`;
                if (branchVisited.has(key)) return false;
                if (terrainMap[ny][nx].name === 'Lake') {
                  return nx === lakeX && ny === lakeY;
                }
                return true;
              });
            
            if (branchNeighbors.length === 0) break;
            
            // Score branch neighbors based on distance to lake
            const scoredBranchNeighbors = branchNeighbors.map(([nx, ny]) => {
              const distanceToLake = Math.abs(nx - lakeX) + Math.abs(ny - lakeY);
              const randomFactor = random() * 1; // Less randomness for branches
              return { pos: [nx, ny], score: distanceToLake + randomFactor };
            });
            
            scoredBranchNeighbors.sort((a, b) => a.score - b.score);
            const [nextX, nextY] = scoredBranchNeighbors[0].pos;
            
            // Set branch river direction
            const direction = getDirectionBetweenHexes(branchX, branchY, nextX, nextY);
            branchHex.riverTo = direction;
            
            const nextHex = hexGrid[nextY * width + nextX];
            nextHex.hasRiver = true;
            nextHex.riverFrom = (direction + 3) % 6;
            
            branchX = nextX;
            branchY = nextY;
            branchVisited.add(`${nextX},${nextY}`);
          }
        }
      }
      
      // Get valid neighbors for main river
      const neighbors = getAdjacentHexes(currentX, currentY)
        .filter(([nx, ny]) => {
          const key = `${nx},${ny}`;
          if (visited.has(key)) return false;
          
          // Special handling for starting position
          if (currentX === startX && currentY === startY) {
            const isCorner = (isStartLeft || isStartRight) && (isStartTop || isStartBottom);
            
            if (isCorner) {
              // When starting from a corner, allow moving diagonally inward
              if (isStartBottom && isStartRight) return ny < currentY || nx < currentX;
              if (isStartBottom && isStartLeft) return ny < currentY || nx > currentX;
              if (isStartTop && isStartRight) return ny > currentY || nx < currentX;
              if (isStartTop && isStartLeft) return ny > currentY || nx > currentX;
            } else {
              // Regular edge handling
              if (isStartBottom) return ny < currentY;
              if (isStartTop) return ny > currentY;
              if (isStartLeft) return nx > currentX;
              if (isStartRight) return nx < currentX;
            }
          } else {
            // Don't allow moving back to the starting edge after the first move
            if (isStartLeft && nx === 0) return false;
            if (isStartRight && nx === width - 1) return false;
            if (isStartTop && ny === 0) return false;
            if (isStartBottom && ny === height - 1) return false;
          }
          
          if (terrainMap[ny][nx].name === 'Lake') {
            return nx === targetX && ny === targetY;
          }
          return true;
        });
      
      // Check if we're at a map edge (but not the starting point) and no valid path to target lake
      const isAtEdge = currentX === 0 || currentX === width - 1 || currentY === 0 || currentY === height - 1;
      const isStartingPoint = currentX === startX && currentY === startY;
      if (isAtEdge && !isStartingPoint && !neighbors.some(([nx, ny]) => nx === targetX && ny === targetY)) {
        console.log('River reached map edge at', { currentX, currentY });
        break;
      }
      
      if (neighbors.length === 0) {
        console.log('River has no valid neighbors at', { currentX, currentY }, 'isStartingPoint:', isStartingPoint);
        break;
      }
      
      // Score neighbors based on distance to target and direction from start
      const scoredNeighbors = neighbors.map(([nx, ny]) => {
        const distanceToTarget = Math.abs(nx - targetX) + Math.abs(ny - targetY);
        const randomFactor = random() * 0.5; // Reduced randomness even more
        const targetPreference = distanceToTarget * 8; // Doubled target preference
        
        // Calculate how much this move gets us closer to the target
        const currentDistanceToTarget = Math.abs(currentX - targetX) + Math.abs(currentY - targetY);
        const distanceImprovement = currentDistanceToTarget - distanceToTarget;
        const progressBonus = distanceImprovement * 12; // Big bonus for getting closer
        
        // Add strong directional preference based on starting edge
        let directionBonus = 0;
        if (isStartLeft) directionBonus = (nx - currentX) * -12; // Increased directional preference
        if (isStartRight) directionBonus = (currentX - nx) * -12;
        if (isStartTop) directionBonus = (ny - currentY) * -12;
        if (isStartBottom) directionBonus = (currentY - ny) * -12;
        
        // Much higher penalty for moving towards any edge unless it's the target
        const isTargetNearEdge = (Math.abs(targetX - nx) <= 1 && (targetY === ny));
        const isMovingTowardsTarget = distanceImprovement > 0;
        const edgePenalty = (nx === 0 || nx === width - 1 || ny === 0 || ny === height - 1) ? 
          (isTargetNearEdge ? 0 : 16) : 0;
        
        // Extra penalty for moving parallel to the starting edge
        const parallelPenalty = 
          (isStartLeft || isStartRight) && (Math.abs(nx - currentX) === 0) ? 8 :
          (isStartTop || isStartBottom) && (Math.abs(ny - currentY) === 0) ? 8 : 0;
        
        return { 
          pos: [nx, ny], 
          score: targetPreference + randomFactor + edgePenalty + directionBonus + parallelPenalty + progressBonus,
          distanceToTarget,
          isMovingTowardsTarget
        };
      });
      
      // Sort by score and prefer moves that get us closer to target
      scoredNeighbors.sort((a, b) => {
        if (a.isMovingTowardsTarget !== b.isMovingTowardsTarget) {
          return a.isMovingTowardsTarget ? -1 : 1;
        }
        return a.score - b.score;
      });
      
      // Pick the best direction with less randomness
      const [nextX, nextY] = scoredNeighbors[0].pos;
      
      // Set river direction
      const direction = getDirectionBetweenHexes(currentX, currentY, nextX, nextY);
      hex.riverTo = direction;
      
      const nextHex = hexGrid[nextY * width + nextX];
      nextHex.hasRiver = true;
      nextHex.riverFrom = (direction + 3) % 6;
      
      currentX = nextX;
      currentY = nextY;
      visited.add(`${nextX},${nextY}`);
    }
    
    if (steps >= maxSteps) {
      console.log('River generation hit maximum steps');
    }
  }

  function generateSettlements(random: () => number): Settlement[] {
    const newSettlements: Settlement[] = [];
    const settlementTypes = Object.keys(SETTLEMENT_TYPES) as (keyof typeof SETTLEMENT_TYPES)[];
    const minDistance = Math.floor(Math.min(width, height) / 2);
    
    function getDistance(x1: number, y1: number, x2: number, y2: number): number {
      return Math.sqrt(Math.pow(x2 - x1, 2) + Math.pow(y2 - y1, 2));
    }
    
    function isValidPosition(x: number, y: number): boolean {
      for (const settlement of newSettlements) {
        if (getDistance(x, y, settlement.x, settlement.y) < minDistance) {
          return false;
        }
      }
      return terrainMap[y][x].name !== 'Lake';
    }
    
    let attempts = 0;
    const maxAttempts = 100;
    
    while (newSettlements.length < 4 && attempts < maxAttempts) {
      attempts++;
      
      const x = Math.floor(random() * width);
      const y = Math.floor(random() * height);
      
      if (isValidPosition(x, y)) {
        const type = settlementTypes[Math.floor(random() * settlementTypes.length)];
        newSettlements.push({ type, x, y });
      }
    }
    
    // Designate seat of power (prefer Castle, then Fortress, then any other)
    const castle = newSettlements.find(s => s.type === 'CASTLE');
    const fortress = newSettlements.find(s => s.type === 'FORTRESS');
    const seatOfPower = castle || fortress || newSettlements[0];
    if (seatOfPower) {
      seatOfPower.seatOfPower = true;
    }
    
    return newSettlements;
  }

  function getHexTooltip(row: number, col: number): string {
    let tooltip = `Terrain: ${terrainMap[row][col].name}`;
    
    // Check for settlement
    const settlement = settlements.find(s => s.x === col && s.y === row);
    if (settlement) {
      tooltip += `\nSettlement: ${SETTLEMENT_TYPES[settlement.type].name}`;
      if (settlement.seatOfPower) tooltip += ' (Seat of Power)';
    }
    
    // Check for myth
    const myth = myths.find(m => m.x === col && m.y === row);
    if (myth) {
      tooltip += `\nMyth: ${myth.name}`;
    }
    
    // Check for landmark
    const landmark = landmarks.find(l => l.x === col && l.y === row);
    if (landmark) {
      tooltip += `\nLandmark: ${LANDMARK_TYPES[landmark.type].name}`;
    }
    
    return tooltip;
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
        <!-- Filled hex with tooltip -->
        <path 
          d={getHexPath(hex.x + hexSize, hex.y + hexSize, [], true)} 
          fill={terrainMap[row][col].color}
          stroke="none"
        >
          <title>{getHexTooltip(row, col)}</title>
        </path>
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
        >
          <title>Barrier</title>
        </path>
      {/if}
    {/each}
    <!-- Draw all rivers -->
    {#each getRiverPoints().split(' M') as riverPath, i}
      <path 
        d={i === 0 ? riverPath : 'M' + riverPath} 
        fill="none"
        stroke={TERRAIN_TYPES.LAKE.color}
        stroke-width="6"
        stroke-linecap="round"
        stroke-linejoin="round"
      >
        <title>River</title>
      </path>
    {/each}
    <!-- Draw all settlements -->
    {#each settlements as settlement}
      <g>
        {#if settlement.seatOfPower}
          <circle
            cx={settlement.x * hexWidth + (settlement.y % 2) * (hexWidth / 2) + hexSize}
            cy={settlement.y * vertOffset + hexSize * 1.5}
            r="22"
            fill="none"
            stroke="red"
            stroke-width="2"
          >
            <title>{`${SETTLEMENT_TYPES[settlement.type].name} (Seat of Power)`}</title>
          </circle>
        {/if}
        <text
          x={settlement.x * hexWidth + (settlement.y % 2) * (hexWidth / 2) + hexSize}
          y={settlement.y * vertOffset + hexSize * 1.5}
          text-anchor="middle"
          font-size="24"
        >
          <title>{`${SETTLEMENT_TYPES[settlement.type].name}${settlement.seatOfPower ? ' (Seat of Power)' : ''}`}</title>
          {SETTLEMENT_TYPES[settlement.type].symbol}
        </text>
      </g>
    {/each}
    <!-- Draw all myths -->
    {#each myths as myth, index}
      <g>
        <circle
          cx={myth.x * hexWidth + (myth.y % 2) * (hexWidth / 2) + hexSize}
          cy={myth.y * vertOffset + hexSize * 1.5}
          r="20"
          fill="none"
          stroke="purple"
          stroke-width="2"
          stroke-dasharray="4,4"
        >
          <title>{myth.name}</title>
        </circle>
        <text
          x={myth.x * hexWidth + (myth.y % 2) * (hexWidth / 2) + hexSize}
          y={myth.y * vertOffset + hexSize * 1.5}
          text-anchor="middle"
          dominant-baseline="central"
          font-size="16"
          fill="purple"
          font-weight="bold"
        >
          <title>{myth.name}</title>
          {index + 1}
        </text>
      </g>
    {/each}
    <!-- Draw all landmarks -->
    {#each landmarks as landmark}
      <g>
        <circle
          cx={landmark.x * hexWidth + (landmark.y % 2) * (hexWidth / 2) + hexSize}
          cy={landmark.y * vertOffset + hexSize * 1.5}
          r="18"
          fill="none"
          stroke={LANDMARK_TYPES[landmark.type].color}
          stroke-width="2"
        >
          <title>{LANDMARK_TYPES[landmark.type].name}</title>
        </circle>
        <text
          x={landmark.x * hexWidth + (landmark.y % 2) * (hexWidth/2) + hexSize}
          y={landmark.y * vertOffset + hexSize * 1.5}
          text-anchor="middle"
          dominant-baseline="central"
          font-size="16"
        >
          <title>{LANDMARK_TYPES[landmark.type].name}</title>
          {LANDMARK_TYPES[landmark.type].symbol}
        </text>
      </g>
    {/each}
  </svg>

  <div class="grid grid-cols-4 gap-2 mt-4">
    {#each Object.values(TERRAIN_TYPES) as terrain}
      <div class="flex items-center">
        <div class="w-4 h-4 mr-2" style="background-color: {terrain.color}"></div>
        <span>{terrain.name}</span>
      </div>
    {/each}
    {#each Object.entries(SETTLEMENT_TYPES) as [type, settlement]}
      <div class="flex items-center">
        <span class="mr-2">{settlement.symbol}</span>
        <span>{settlement.name}</span>
      </div>
    {/each}
    <div class="col-span-4 mt-2">
      <h3 class="font-bold mb-2">Myths:</h3>
      <div class="grid grid-cols-2 gap-2">
        {#each myths as myth, index}
          <div class="flex items-center">
            <span class="mr-2 text-purple-600 font-bold">{index + 1}</span>
            <span>{myth.name}</span>
          </div>
        {/each}
      </div>
    </div>
    <div class="col-span-4 mt-2">
      <h3 class="font-bold mb-2">Landmarks:</h3>
      <div class="grid grid-cols-3 gap-2">
        {#each Object.entries(LANDMARK_TYPES) as [type, landmark]}
          <div class="flex items-center">
            <span class="mr-2" style="color: {landmark.color}">{landmark.symbol}</span>
            <span>{landmark.name}</span>
          </div>
        {/each}
      </div>
    </div>
  </div>
</div>
