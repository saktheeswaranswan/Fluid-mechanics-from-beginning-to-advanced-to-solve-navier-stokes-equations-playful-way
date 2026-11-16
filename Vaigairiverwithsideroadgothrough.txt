/*
VAIGAI RIVER BRIDGE - MADURAI, TAMIL NADU 3D FLUID MECHANICS & TRAFFIC SIMULATION 
====================================================================

PLATFORM:
PROCESSING ANDROID - APDE / JAVA MODE

RENDERER:
P3D

FLUID MECHANICS VISUALIZATION:
1. STREAMLINES (CYAN) : Instantaneous tangent curves to velocity field
2. STREAKLINES (MAGENTA) : Locus of particles passed through fixed injectors
3. PATHLINES (GOLD) : Actual Lagrangian trajectories of individual particles
4. FLOW LINES & VELOCITY PARTICLES : 
   - Tiny spheres representing fluid parcels with exact (vx, vz) velocity vectors
   - Velocity Magnitude Color Map:
     BLUE (Low Velocity) -> CYAN/GREEN -> YELLOW -> ORANGE -> RED (High Velocity)
   - Attached 3D direction vector arrows scaled & colored by velocity magnitude

ROADS AT BOTH SIDES OF BRIDGE & UNDER-ARCH TRAFFIC:
- RIVERBANK ROADS (BOTH SIDES): Run straight out vertically (perpendicular to
  the bridge) right between the last two pillars and through the last parabolic
  arch at both ends of the bridge, with live two-way traffic
- BRIDGE APPROACH ROADS & RAMPS (BOTH ENDS): Graded embankments and paved shore
  avenues connecting both ends of the bridge deck smoothly to the shore roads
- SUPERBIKE RACER & 3D CAD DIMENSION LINES: Calculates exact inner-edge gap
  between flanking vehicles, locks onto midpoint, and threads cleanly through

CONTROLS:
DRAG BIG SLIDERS = SMOOTHLY ADJUST PARABOLIC ARCH DISTANCE & ARCH RISE
TAP ANY VEHICLE / RACER = OPEN LIVE ANALOG + DIGITAL SPEEDOMETER DIAL
DRAG 3D SCENE = ROTATE 3D CAMERA
TAP EMPTY SCENE = PAUSE / RESUME SIMULATION
TAP BUTTONS = TOGGLE VISUALIZATIONS, SPEEDS, VECTORS, GAP DIMS & FLOW REGIMES
====================================================================
*/

// ============================================================
// GLOBAL VARIABLES
// ============================================================

boolean simulationRunning = true;
float sceneTime = 0;

// Camera settings (framed to show bridge, both sides, and end arches)
float cameraYaw = 0.14;
float cameraPitch = 0.44;
float cameraDistance = 1720;

float previousMouseX;
float previousMouseY;
boolean dragging = false;

// ============================================================
// FLUID MECHANICS & VEHICLE TELEMETRY TOGGLES
// ============================================================

boolean showStreamlines = true;
boolean showStreaklines = true;
boolean showPathlines = true;
boolean showFlowlines = true;
boolean showAllVehicleSpeeds = true;
boolean showVehicleVectors = true;
boolean showGapDimensions = true;

// 0 = STEADY FLOW, 1 = LAMINAR (STREAMLINE) FLOW, 2 = TURBULENT FLOW
int flowRegime = 1;

// HUD Button layout arrays (10 interactive buttons across 3 rows)
String[] btnLabels = {
  "STREAMLINES",
  "STREAKLINES",
  "PATHLINES",
  "PARTICLE VECTORS",
  "STEADY FLOW",
  "LAMINAR FLOW",
  "TURBULENT FLOW",
  "ALL SPEEDS",
  "VEH VECTORS",
  "GAP DIMS"
};
float[] btnX = new float[10];
float[] btnY = new float[10];
float[] btnW = new float[10];
float[] btnH = new float[10];

// Large Smooth Interactive Sliders (0 = Parabolic Arch Distance, 1 = Parabolic Arch Rise)
int activeSlider = -1;
float[] sliderX = new float[2];
float[] sliderY = new float[2];
float[] sliderW = new float[2];
float[] sliderH = new float[2];
float[] sliderNorm = new float[2];

float minArchDistance = 85.0;        // 1700 / 20 arches = 85.0 m
float maxArchDistance = 283.3333;    // 1700 / 6 arches  = 283.3 m
float targetArchDistance = 113.3333; // 1700 / 15 arches = 113.3 m

float minArchRise = 65.0;
float maxArchRise = 132.0;

// ============================================================
// BRIDGE, APPROACH ROADS & RIVERBANK UNDERPASS DIMENSIONS
// ============================================================

float bridgeLength = 1700;
float bridgeWidth = 380;
float roadY = 78;
int archCount = 15;
float spanLength = bridgeLength / float(archCount);
float archSpringY = 278;
float archRise = 118;
float archThickness = 18;

// Shore & Approach Road Geometry
float shoreGroundY = 231.5;
float shoreRoadSurfaceY = 226.5;
float bridgeRampStart = 940.0;
float bridgeRampEnd = 1440.0;
float approachEnd = 1660.0;

// Riverbank Underpass Road (passes straight out vertically through last parabola)
float bankRoadSurfaceY = 266.0;
float bankRoadHalfLength = 1280.0;

// ============================================================
// RIVER & SHORE DIMENSIONS
// ============================================================

float waterLevel = 310;
float riverWidth = 1700;
float riverLength = 2600;

// ============================================================
// HYDRODYNAMIC FLOW ARRAYS
// ============================================================

// 1. Streamlines (Cyan)
int streamLineCount = 24;

// 2. Velocity Vector Particles (Tiny spheres with magnitude & direction vectors)
int flowParticleCount = 260;
float[] flowPartX = new float[260];
float[] flowPartZ = new float[260];
PVector[] flowPartVel = new PVector[260];
float[] flowPartMag = new float[260];

// Velocity color scale bounds
float minSpeedScale = 1.2;
float maxSpeedScale = 6.4;

// 3. Streaklines (Magenta/Pink with highlighted point spheres)
int streakEmitterCount = 8;
int streakTrailLength = 36;
float[][] streakX = new float[8][36];
float[][] streakZ = new float[8][36];
float[] emitterZ = new float[8];

// 4. Pathlines (Gold/Orange Lagrangian particle histories with tracer spheres)
int pathParticleCount = 14;
int pathHistoryLength = 55;
float[][] pathX = new float[14][55];
float[][] pathZ = new float[14][55];
int[] pathCount = new int[14];
float[] pathSpeedFactor = new float[14];

// ============================================================
// MAIN BRIDGE VEHICLE ARRAYS & SPEEDOMETER TELEMETRY
// ============================================================

int vehicleCount = 36;
float[] vehiclePosition = new float[36];
float[] vehicleBaseSpeed = new float[36];
float[] vehicleSpeed = new float[36];
float[] vehicleLane = new float[36];
int[] vehicleDirection = new int[36];
int[] vehicleType = new int[36];
color[] vehicleColors = new color[36];

// 2D Screen projections for touch selection and live speed/vector overlays
float[] vehScreenX = new float[36];
float[] vehScreenY = new float[36];
float[] vecTipScreenX = new float[36];
float[] vecTipScreenY = new float[36];
boolean[] vehVisible = new boolean[36];

// ============================================================
// RIVERBANK UNDERPASS ROAD TRAFFIC (THROUGH LAST PARABOLA ON BOTH SIDES)
// ============================================================

int RACER_ID = 36;
int BANK_VEH_ID_OFFSET = 37;
int bankVehicleCount = 24; // 12 on South/Left Bank + 12 on North/Right Bank

int[] bankVehSide = new int[24];          // -1 = Left/South Bank, +1 = Right/North Bank
float[] bankVehZ = new float[24];         // Position along perpendicular bank road
float[] bankVehLaneSide = new float[24];  // -1 = Westbound lane, +1 = Eastbound lane
int[] bankVehDir = new int[24];           // -1 or +1 along localZ
float[] bankVehBaseSpeed = new float[24];
float[] bankVehSpeed = new float[24];
int[] bankVehType = new int[24];
color[] bankVehColors = new color[24];

float[] bankVehScreenX = new float[24];
float[] bankVehScreenY = new float[24];
float[] bankVecTipScreenX = new float[24];
float[] bankVecTipScreenY = new float[24];
boolean[] bankVehVisible = new boolean[24];

// ============================================================
// HIGH-SPEED SPORTBIKE RACER & LIVE GAP DIMENSION TELEMETRY
// ============================================================

float RACER_HALF_WIDTH = 6.0; // 12.0 units total width = 1.20 m envelope
float UNIT_TO_METERS = 0.1;   // 10 simulation units = 1.00 m lateral/gap scale

float racerX = -650;
float racerZ = 91;
float racerTargetZ = 91;
float racerVx = 6.3;
float racerVz = 0.0;
float racerSpeed = 6.3;
float racerBaseSpeed = 6.5;
float racerLeanAngle = 0.0;
float racerYawAngle = 0.0;

// Live calculated gap geometry between vehicles for CAD dimension lines
boolean racerGapActive = false;
int racerLeftVehIdx = -1;
int racerRightVehIdx = -1;
float racerGapLeftEdgeZ = 78.0;
float racerGapRightEdgeZ = 104.0;
float racerGapDimX = -500.0;
float racerCalculatedGapUnits = 26.0;
float racerCalculatedMidZ = 91.0;
float racerLeftClearUnits = 7.0;
float racerRightClearUnits = 7.0;
boolean racerGapPassable = true;

// 2D Screen coordinates for active gap dimension badges
float gapDimScreenX = 0;
float gapDimScreenY = 0;
boolean gapDimVisible = false;

float subGapScreenX = 0;
float subGapScreenY = 0;
boolean subGapVisible = false;

// 2D Screen coordinates for traffic inter-vehicle spacing dimension labels
int MAX_TRAFFIC_DIMS = 48;
int trafficDimCount = 0;
float[] trafficDimScreenX = new float[48];
float[] trafficDimScreenY = new float[48];
float[] trafficDimMeters = new float[48];
boolean[] trafficDimIsLateral = new boolean[48];

float racerScreenX = 0;
float racerScreenY = 0;
float racerVecTipScreenX = 0;
float racerVecTipScreenY = 0;
boolean racerVisible = false;

int racerTrailLength = 26;
float[] racerTrailX = new float[26];
float[] racerTrailZ = new float[26];
int racerTrailCount = 0;

// Selected vehicle speedometer state (-1 = none, 0..35 = bridge traffic, 36 = bike racer, 37..60 = bank road traffic)
int selectedVehicle = -1;
float displayedDialKmh = 0;
float speedToKmhScale = 20.0;

// Speedometer HUD card bounds for touch handling
float dialCardX, dialCardY, dialCardW, dialCardH;
float dialCloseX, dialCloseY, dialCloseR;

// ============================================================
// BUILDING COLORS
// ============================================================

color[] buildingColors = {
  color(220, 205, 175),
  color(195, 185, 160),
  color(230, 215, 190),
  color(185, 175, 155),
  color(205, 195, 180),
  color(220, 200, 170),
  color(180, 185, 175)
};

// ============================================================
// SETTINGS
// ============================================================

void settings() {
  fullScreen(P3D);
}

// ============================================================
// SETUP
// ============================================================

void setup() {
  smooth(4);
  frameRate(40);
  updateBridgeArchSpacing(targetArchDistance);
  sliderNorm[0] = constrain((spanLength - minArchDistance) / (maxArchDistance - minArchDistance), 0.0, 1.0);
  sliderNorm[1] = constrain((archRise - minArchRise) / (maxArchRise - minArchRise), 0.0, 1.0);
  setupControls();
  initializeVehicles();
  initializeBankVehicles();
  initializeBikeRacer();
  initializeRiverFlow();
}

// ============================================================
// BRIDGE & UNDER-ARCH ROAD GEOMETRY HELPERS
// ============================================================

void updateBridgeArchSpacing(float requestedDistance) {
  targetArchDistance = constrain(requestedDistance, minArchDistance, maxArchDistance);
  archCount = constrain(round(bridgeLength / targetArchDistance), 6, 20);
  spanLength = bridgeLength / float(archCount);
}

float getPierWidth() {
  return min(42.0, spanLength * 0.32);
}

float getClearArchOpening() {
  return spanLength - getPierWidth();
}

// Exact center X (bridge local coords) between the last two pillars at each side
float getBankRoadCenterX(int side) {
  if (side < 0) {
    return -bridgeLength * 0.5 + spanLength * 0.5; // Midpoint between Pier 0 and Pier 1
  } else {
    return bridgeLength * 0.5 - spanLength * 0.5;  // Midpoint between Pier (N-1) and Pier N
  }
}

// Dynamic width of each perpendicular bank road so it fits cleanly between the last two pillars
float getBankRoadWidth() {
  float clearOpening = getClearArchOpening();
  return min(132.0, clearOpening * 0.86);
}

// Vehicle scale factor on the riverbank roads so traffic threads cleanly through the last parabola
float getBankVehicleScale() {
  float roadW = getBankRoadWidth();
  return constrain((roadW * 0.44) / 48.0, 0.42, 0.88);
}

// Active open-water Z half-width between the two stone riverbank quays
float getActiveWaterHalfWidth() {
  return max(260.0, riverWidth * 0.5 - spanLength - 14.0);
}

// Vertical Y profile along the bridge deck + sloped shore approach roads at both sides
float getBridgeRoadSurfaceY(float localX) {
  float ax = abs(localX);
  float deckTopY = roadY - 5.0;
  if (ax <= bridgeRampStart) {
    return deckTopY;
  } else if (ax <= bridgeRampEnd) {
    float t = (ax - bridgeRampStart) / (bridgeRampEnd - bridgeRampStart);
    float smoothT = t * t * (3.0 - 2.0 * t);
    return lerp(deckTopY, shoreRoadSurfaceY, smoothT);
  } else {
    return shoreRoadSurfaceY;
  }
}

// Pitch angle along the bridge approach ramps
float getBridgeRoadPitch(float localX, int dir) {
  float eps = 6.0;
  float y1 = getBridgeRoadSurfaceY(localX - eps);
  float y2 = getBridgeRoadSurfaceY(localX + eps);
  float dy = (y2 - y1) * dir;
  return atan2(-dy, 2.0 * eps);
}

// ============================================================
// SETUP RESPONSIVE HUD BUTTONS & LARGE FULL-WIDTH SLIDERS
// ============================================================

void setupControls() {
  float margin = 16;
  float gap = 8;
  float usableW = width - margin * 2;
  float bh = 40;

  // Row 1: Fluid Visualization Line Toggles (4 buttons)
  float bw = (usableW - gap * 3) / 4.0;
  float row1Y = 132;
  for (int i = 0; i < 4; i++) {
    btnX[i] = margin + i * (bw + gap);
    btnY[i] = row1Y;
    btnW[i] = bw;
    btnH[i] = bh;
  }

  // Row 2: Flow Regime Selectors (3 buttons)
  float row2Y = row1Y + bh + 8;
  float rw = (usableW - gap * 2) / 3.0;
  for (int i = 0; i < 3; i++) {
    btnX[4 + i] = margin + i * (rw + gap);
    btnY[4 + i] = row2Y;
    btnW[4 + i] = rw;
    btnH[4 + i] = bh;
  }

  // Row 3: Vehicle Speed, Vector & Gap Dimension Toggles (3 buttons)
  float row3Y = row2Y + bh + 8;
  float vw = (usableW - gap * 2) / 3.0;
  for (int i = 0; i < 3; i++) {
    btnX[7 + i] = margin + i * (vw + gap);
    btnY[7 + i] = row3Y;
    btnW[7 + i] = vw;
    btnH[7 + i] = bh;
  }

  // Rows 4 & 5: Large Full-Width Sliders stacked cleanly for smooth finger control
  float sh = 70;
  float sliderStartY = row3Y + bh + 10;
  for (int s = 0; s < 2; s++) {
    sliderX[s] = margin;
    sliderY[s] = sliderStartY + s * (sh + 8);
    sliderW[s] = usableW;
    sliderH[s] = sh;
  }

  // Bottom Speedometer Card Geometry
  dialCardW = min(width - 32, 540);
  dialCardH = 192;
  dialCardX = (width - dialCardW) * 0.5;
  dialCardY = height - dialCardH - 18;
  dialCloseR = 18;
  dialCloseX = dialCardX + dialCardW - 26;
  dialCloseY = dialCardY + 24;
}

// ============================================================
// INITIALIZE VEHICLES (MAIN BRIDGE + RIVERBANK UNDERPASS ROADS)
// ============================================================

void initializeVehicles() {
  float[] lanes = {-128, -54, 54, 128};
  float fullTrack = approachEnd * 2.0;

  for (int i = 0; i < vehicleCount; i++) {
    int laneIndex = i % 4;
    vehicleLane[i] = lanes[laneIndex];
    vehicleDirection[i] = (vehicleLane[i] > 0) ? 1 : -1;

    int slotInLane = i / 4;
    int vehiclesPerLane = vehicleCount / 4;
    float spacing = fullTrack / vehiclesPerLane;
    vehiclePosition[i] = -approachEnd + slotInLane * spacing + random(-24, 24);
    vehicleBaseSpeed[i] = random(2.1, 3.5);
    vehicleSpeed[i] = vehicleBaseSpeed[i];
    vehicleType[i] = int(random(5));

    int c = int(random(6));
    if (c == 0) vehicleColors[i] = color(240, 235, 220);
    if (c == 1) vehicleColors[i] = color(195, 35, 25);
    if (c == 2) vehicleColors[i] = color(35, 65, 110);
    if (c == 3) vehicleColors[i] = color(220, 175, 30);
    if (c == 4) vehicleColors[i] = color(55, 55, 55);
    if (c == 5) vehicleColors[i] = color(170, 175, 180);
  }
}

void initializeBankVehicles() {
  int perSide = bankVehicleCount / 2; // 12 vehicles per riverbank road
  int perLane = perSide / 2;          // 6 vehicles per direction
  float fullBankTrack = bankRoadHalfLength * 2.0;
  float spacing = fullBankTrack / float(perLane);

  for (int i = 0; i < bankVehicleCount; i++) {
    int side = (i < perSide) ? -1 : 1;
    int localIdx = i % perSide;
    int laneDir = (localIdx % 2 == 0) ? 1 : -1;
    int slot = localIdx / 2;

    bankVehSide[i] = side;
    bankVehDir[i] = laneDir;
    bankVehLaneSide[i] = (laneDir == 1) ? 1.0 : -1.0;
    bankVehZ[i] = -bankRoadHalfLength + slot * spacing + random(-35, 35);
    bankVehBaseSpeed[i] = random(2.0, 3.4);
    bankVehSpeed[i] = bankVehBaseSpeed[i];

    // Mix of Sedans, Auto Rickshaws, Cargo Vans & Commuter Bikes for crisp under-arch passage
    int[] allowedTypes = {0, 1, 3, 4, 0, 1};
    bankVehType[i] = allowedTypes[i % allowedTypes.length];

    int c = int(random(6));
    if (c == 0) bankVehColors[i] = color(245, 240, 225);
    if (c == 1) bankVehColors[i] = color(210, 45, 35);
    if (c == 2) bankVehColors[i] = color(35, 85, 145);
    if (c == 3) bankVehColors[i] = color(235, 185, 35);
    if (c == 4) bankVehColors[i] = color(65, 70, 75);
    if (c == 5) bankVehColors[i] = color(40, 155, 115);
  }
}

void initializeBikeRacer() {
  racerX = -520;
  racerZ = 91;
  racerTargetZ = 91;
  racerBaseSpeed = 6.4;
  racerVx = racerBaseSpeed;
  racerVz = 0;
  racerSpeed = racerBaseSpeed;
  racerLeanAngle = 0;
  racerYawAngle = 0;
  racerTrailCount = 0;
}

// ============================================================
// INITIALIZE RIVER FLOW (PARTICLE VECTORS, STREAKLINES, PATHLINES)
// ============================================================

void initializeRiverFlow() {
  float waterZLimit = getActiveWaterHalfWidth() - 18.0;

  for (int i = 0; i < flowParticleCount; i++) {
    flowPartX[i] = random(-riverLength / 2, riverLength / 2);
    flowPartZ[i] = random(-waterZLimit, waterZLimit);
    PVector initialVel = getRiverVelocity(flowPartX[i], flowPartZ[i], sceneTime);
    flowPartVel[i] = initialVel.copy();
    flowPartMag[i] = initialVel.mag();
  }

  for (int e = 0; e < streakEmitterCount; e++) {
    float zBase = map(e, 0, streakEmitterCount - 1, -waterZLimit + 45, waterZLimit - 45);
    emitterZ[e] = zBase;
    for (int k = 0; k < streakTrailLength; k++) {
      streakX[e][k] = -riverLength / 2 + 70;
      streakZ[e][k] = zBase;
    }
  }

  for (int p = 0; p < pathParticleCount; p++) {
    resetPathParticle(p, true);
  }

  for (int step = 0; step < 360; step++) {
    updateStreaklines(sceneTime + step * 0.025);
    updatePathlines(sceneTime + step * 0.025);
  }
}

void resetPathParticle(int p, boolean staggerX) {
  float waterZLimit = getActiveWaterHalfWidth() - 35.0;
  float startX = staggerX 
    ? map(p, 0, pathParticleCount - 1, -riverLength / 2 + 80, 100) 
    : -riverLength / 2 + 60;
  float startZ = map(p, 0, pathParticleCount - 1, -waterZLimit, waterZLimit);

  pathX[p][0] = startX;
  pathZ[p][0] = startZ;
  pathCount[p] = 1;
  pathSpeedFactor[p] = 1.0 + (p % 3) * 0.08;
}

// ============================================================
// VELOCITY MAGNITUDE COLOR MAPPER
// ============================================================

color getVelocityColor(float speed, float alpha) {
  float norm = constrain((speed - minSpeedScale) / (maxSpeedScale - minSpeedScale), 0.0, 1.0);

  color cLowBlue   = color(20, 95, 255);
  color cMidCyan   = color(25, 215, 210);
  color cYellow    = color(255, 235, 30);
  color cOrange    = color(255, 130, 10);
  color cHighRed   = color(255, 25, 25);

  color result;
  if (norm < 0.25) {
    result = lerpColor(cLowBlue, cMidCyan, norm / 0.25);
  } else if (norm < 0.55) {
    result = lerpColor(cMidCyan, cYellow, (norm - 0.25) / 0.30);
  } else if (norm < 0.80) {
    result = lerpColor(cYellow, cOrange, (norm - 0.55) / 0.25);
  } else {
    result = lerpColor(cOrange, cHighRed, (norm - 0.80) / 0.20);
  }

  return color(red(result), green(result), blue(result), alpha);
}

color getVehicleSpeedColor(float spd, float alpha) {
  float norm = constrain((spd - 0.2) / 6.5, 0.0, 1.0);
  color cSlow   = color(60, 175, 255);
  color cMed    = color(75, 235, 135);
  color cCruise = color(255, 215, 45);
  color cFast   = color(255, 75, 55);

  color result;
  if (norm < 0.30) {
    result = lerpColor(cSlow, cMed, norm / 0.30);
  } else if (norm < 0.58) {
    result = lerpColor(cMed, cCruise, (norm - 0.30) / 0.28);
  } else {
    result = lerpColor(cCruise, cFast, (norm - 0.58) / 0.42);
  }
  return color(red(result), green(result), blue(result), alpha);
}

String getVehicleTypeName(int type) {
  if (type == 0) return "SEDAN CAR";
  if (type == 1) return "AUTO RICKSHAW";
  if (type == 2) return "CITY BUS";
  if (type == 3) return "CARGO LORRY";
  if (type == 4) return "COMMUTER BIKE";
  return "SUPERBIKE RACER";
}

// ============================================================
// MAIN DRAW
// ============================================================

void draw() {
  if (simulationRunning) {
    sceneTime += 0.025;
    updateRiverFlow();
  }

  // Smoothly align slider 0 thumb to the exact spanLength once finger releases
  if (activeSlider != 0) {
    float exactNorm = constrain((spanLength - minArchDistance) / (maxArchDistance - minArchDistance), 0.0, 1.0);
    sliderNorm[0] = lerp(sliderNorm[0], exactNorm, 0.22);
  }

  // Smoothly update speedometer dial needle for selected vehicle or bike racer
  if (selectedVehicle >= 0 && selectedVehicle < vehicleCount) {
    float targetKmh = vehicleSpeed[selectedVehicle] * speedToKmhScale;
    displayedDialKmh = lerp(displayedDialKmh, targetKmh, 0.22);
  } else if (selectedVehicle == RACER_ID) {
    float targetKmh = racerSpeed * speedToKmhScale;
    displayedDialKmh = lerp(displayedDialKmh, targetKmh, 0.25);
  } else if (selectedVehicle >= BANK_VEH_ID_OFFSET && selectedVehicle < BANK_VEH_ID_OFFSET + bankVehicleCount) {
    int bIdx = selectedVehicle - BANK_VEH_ID_OFFSET;
    float targetKmh = bankVehSpeed[bIdx] * speedToKmhScale;
    displayedDialKmh = lerp(displayedDialKmh, targetKmh, 0.22);
  }

  background(158, 188, 212);

  setCamera();
  drawLighting();
  drawLongShoreBanks();
  drawShoreBuildings();
  drawRiver();

  drawRiverHydrodynamics();

  pushMatrix();
  rotateY(HALF_PI);
  drawSideBankRoadsAndApproaches();
  drawBridgeSupports();
  drawSeparatedParabolicArches();
  drawBridgeArchDimensionCallout();
  drawBridgeDeck();
  drawBridgeRailings();
  drawStreetLights();
  drawVehicles();
  drawBankRoadVehicles();
  drawBikeRacer();
  drawVehicleDimensionLines();
  popMatrix();

  drawWaterFoam();
  drawInformation();
}

// ============================================================
// CAMERA
// ============================================================

void setCamera() {
  float targetX = 0;
  float targetY = 175;
  float targetZ = 0;

  float cx = cos(cameraYaw) * cos(cameraPitch) * cameraDistance;
  float cy = targetY - sin(cameraPitch) * cameraDistance;
  float cz = sin(cameraYaw) * cos(cameraPitch) * cameraDistance;

  camera(
    targetX + cx, cy, targetZ + cz,
    targetX, targetY, targetZ,
    0, 1, 0
  );

  perspective(PI / 2.8, float(width) / float(height), 10, 8500);
}

// ============================================================
// LIGHTING
// ============================================================

void drawLighting() {
  ambientLight(145, 145, 150);
  directionalLight(245, 230, 210, -0.5, 0.8, -0.4);
}

float randomStable(int n) {
  float v = sin(n * 127.1 + 311.7) * 43758.5453;
  return v - floor(v);
}

// ============================================================
// SHORES & BUILDINGS
// ============================================================

void drawLongShoreBanks() {
  for (int side = -1; side <= 1; side += 2) {
    pushMatrix();
    translate(0, 275, side * (riverWidth / 2 + 450));
    noStroke();
    fill(135, 110, 80);
    box(riverLength + 300, 65, 900);

    pushMatrix();
    translate(0, -36, 0);
    fill(95, 115, 65);
    box(riverLength + 260, 15, 880);
    popMatrix();
    popMatrix();
  }
}

void drawShoreBuildings() {
  for (int side = -1; side <= 1; side += 2) {
    for (int i = 0; i < 35; i++) {
      float x = -riverLength / 2 + i * 140;
      // Leave clear avenue corridor around worldX = [-230, +230] for the bridge approach road
      if (abs(x) < 250) continue;

      float z = side * (960 + randomStable(i * 11 + side * 5) * 480);
      float h = 70 + randomStable(i * 19) * 140;
      float w = 80 + randomStable(i * 23) * 50;

      pushMatrix();
      translate(x, 230 - h / 2, z);
      noStroke();
      fill(buildingColors[i % buildingColors.length]);
      box(w, h, 95);
      drawShoreWindows(w, h, 95);
      popMatrix();
    }
  }
}

void drawShoreWindows(float w, float h, float d) {
  for (int row = 0; row < 4; row++) {
    for (int col = 0; col < 3; col++) {
      pushMatrix();
      float wx = -w * 0.3 + col * w * 0.3;
      float wy = -h * 0.3 + row * h * 0.2;
      translate(wx, wy, d / 2 + 1);
      noStroke();
      fill(70, 110, 130);
      box(12, 16, 2);
      popMatrix();
    }
  }
}

// ============================================================
// ROADS AT BOTH SIDES OF BRIDGE:
// 1. RIVERBANK UNDERPASS ROADS GOING STRAIGHT OUT VERTICALLY
//    THROUGH THE LAST PARABOLIC ARCH IN BETWEEN THE LAST TWO PILLARS
// 2. GRADED BRIDGE APPROACH ROADS & EMBANKMENTS ON BOTH SIDES
// ============================================================

void drawSideBankRoadsAndApproaches() {
  float pierW = getPierWidth();
  float bankRoadW = getBankRoadWidth();
  float bankRoadLen = bankRoadHalfLength * 2.0;

  // ------------------------------------------------------------------
  // 1. RIVERBANK ROADS AT BOTH SIDES (THROUGH LAST PARABOLIC ARCH)
  //    Centered exactly at midpoint between the last two pillars
  //    Running straight out vertically (along localZ / worldX)
  // ------------------------------------------------------------------
  for (int side = -1; side <= 1; side += 2) {
    float roadCenterX = getBankRoadCenterX(side);
    float outerPierX = side * (bridgeLength * 0.5);
    float innerPierX = side * (bridgeLength * 0.5 - spanLength);
    float quayMinX = min(outerPierX, innerPierX);
    float quayMaxX = max(outerPierX, innerPierX);
    float quaySpanX = max(10.0, quayMaxX - quayMinX);
    float quayCenterX = (quayMinX + quayMaxX) * 0.5;

    // A) Stone Riverbank Quay & Retaining Embankment between the last two pillars
    pushMatrix();
    translate(quayCenterX, 303.0, 0);
    noStroke();
    fill(132, 118, 98);
    box(quaySpanX + 6, 74, bankRoadLen);

    // River-side dressed stone retaining parapet wall along inner pier line
    pushMatrix();
    translate(-side * (quaySpanX * 0.5 - 3), -40, 0);
    fill(175, 165, 145);
    box(8, 14, bankRoadLen);
    popMatrix();

    // Shore-side stepped retaining terrace connecting shore bank (y=231.5) to bank road (y=266)
    pushMatrix();
    translate(side * (quaySpanX * 0.5 + 8), -53, 0);
    fill(145, 128, 104);
    box(20, 42, bankRoadLen);
    popMatrix();
    popMatrix();

    // B) Asphalt Riverbank Underpass Road Deck (going straight out vertically along localZ)
    pushMatrix();
    translate(roadCenterX, bankRoadSurfaceY + 2.5, 0);
    noStroke();
    fill(48, 50, 52);
    box(bankRoadW, 5, bankRoadLen);

    // Raised concrete road curbs on both edges (staying well inside the two pillars)
    fill(205, 200, 188);
    pushMatrix();
    translate(-bankRoadW * 0.5 + 2.2, -3.5, 0);
    box(4.4, 4, bankRoadLen);
    popMatrix();

    pushMatrix();
    translate(bankRoadW * 0.5 - 2.2, -3.5, 0);
    box(4.4, 4, bankRoadLen);
    popMatrix();
    popMatrix();

    // C) Double Yellow Centerline & Dashed White Lane Markings along the Bank Road
    float markY = bankRoadSurfaceY - 0.4;
    stroke(240, 195, 55);
    strokeWeight(2.4);
    line(roadCenterX - 1.4, markY, -bankRoadHalfLength + 20, roadCenterX - 1.4, markY, bankRoadHalfLength - 20);
    line(roadCenterX + 1.4, markY, -bankRoadHalfLength + 20, roadCenterX + 1.4, markY, bankRoadHalfLength - 20);

    // Subtle emerald guide stripes showing the clear corridor between the last two pillars
    stroke(65, 235, 165, 150);
    strokeWeight(1.8);
    float edgeStripe = bankRoadW * 0.43;
    line(roadCenterX - edgeStripe, markY, -bankRoadHalfLength + 20, roadCenterX - edgeStripe, markY, bankRoadHalfLength - 20);
    line(roadCenterX + edgeStripe, markY, -bankRoadHalfLength + 20, roadCenterX + edgeStripe, markY, bankRoadHalfLength - 20);

    // D) Bank Road Streetlights along the outer edges (outside the bridge deck footprint)
    for (int k = -11; k <= 11; k++) {
      float lz = k * 110.0;
      if (abs(lz) < bridgeWidth * 0.5 + 36.0) continue; // Keep clear under the bridge deck

      float poleX = roadCenterX + side * (bankRoadW * 0.5 + 6.0);
      pushMatrix();
      translate(poleX, bankRoadSurfaceY, lz);
      noStroke();
      fill(70, 72, 70);
      pushMatrix();
      translate(0, -48, 0);
      box(4, 96, 4);
      popMatrix();

      pushMatrix();
      translate(-side * 12, -96, 0);
      rotateZ(-side * 0.35);
      box(26, 3, 3);
      popMatrix();

      fill(255, 235, 175);
      pushMatrix();
      translate(-side * 23, -101, 0);
      sphereDetail(6);
      sphere(5.5);
      popMatrix();
      popMatrix();
    }
  }

  // ------------------------------------------------------------------
  // 2. MAIN BRIDGE APPROACH ROADS & GRADED EMBANKMENTS (BOTH ENDS)
  //    Connects bridge deck (y=78) smoothly down to shore avenues (y=226.5)
  // ------------------------------------------------------------------
  int rampSegs = 28;
  float startX = bridgeRampStart;
  float endX = approachEnd;
  float halfW = bridgeWidth * 0.5;
  float asphaltHalfW = (bridgeWidth - 35.0) * 0.5;

  for (int side = -1; side <= 1; side += 2) {
    noStroke();

    // Solid masonry/earth embankment wall under the sloped approach road
    fill(152, 112, 80);
    beginShape(QUAD_STRIP);
    for (int s = 0; s <= rampSegs; s++) {
      float ax = lerp(startX, endX, float(s) / rampSegs);
      float x = side * ax;
      float topY = getBridgeRoadSurfaceY(x) + 4.0;
      float baseY = shoreGroundY + 12.0;
      vertex(x, topY, -halfW);
      vertex(x, baseY, -halfW);
    }
    endShape();

    beginShape(QUAD_STRIP);
    for (int s = 0; s <= rampSegs; s++) {
      float ax = lerp(startX, endX, float(s) / rampSegs);
      float x = side * ax;
      float topY = getBridgeRoadSurfaceY(x) + 4.0;
      float baseY = shoreGroundY + 12.0;
      vertex(x, topY, halfW);
      vertex(x, baseY, halfW);
    }
    endShape();

    // End cap wall at approachEnd
    fill(140, 102, 72);
    beginShape(QUADS);
    float capX = side * endX;
    float capTopY = getBridgeRoadSurfaceY(capX) + 4.0;
    vertex(capX, capTopY, -halfW);
    vertex(capX, capTopY, halfW);
    vertex(capX, shoreGroundY + 12.0, halfW);
    vertex(capX, shoreGroundY + 12.0, -halfW);
    endShape();

    // Graded Asphalt Road Surface along the approach ramp & shore avenue
    fill(55, 55, 53);
    beginShape(QUAD_STRIP);
    for (int s = 0; s <= rampSegs; s++) {
      float ax = lerp(startX, endX, float(s) / rampSegs);
      float x = side * ax;
      float surfY = getBridgeRoadSurfaceY(x);
      vertex(x, surfY, -asphaltHalfW);
      vertex(x, surfY, asphaltHalfW);
    }
    endShape();

    // Concrete Side Parapet Curbs & Guardrails along both sides of the approach road
    for (int railSide = -1; railSide <= 1; railSide += 2) {
      float rz = railSide * (bridgeWidth * 0.5 - 10.0);
      fill(208, 203, 188);
      beginShape(QUAD_STRIP);
      for (int s = 0; s <= rampSegs; s++) {
        float ax = lerp(startX, endX, float(s) / rampSegs);
        float x = side * ax;
        float surfY = getBridgeRoadSurfaceY(x);
        vertex(x, surfY - 26, rz);
        vertex(x, surfY + 6, rz);
      }
      endShape();
    }

    // Double Yellow Centerline & Dashed White Lane Lines along the approach ramp
    for (int s = 0; s < rampSegs; s++) {
      float ax1 = lerp(startX, endX, float(s) / rampSegs);
      float ax2 = lerp(startX, endX, float(s + 1) / rampSegs);
      float x1 = side * ax1;
      float x2 = side * ax2;
      float y1 = getBridgeRoadSurfaceY(x1) - 0.6;
      float y2 = getBridgeRoadSurfaceY(x2) - 0.6;

      stroke(240, 195, 55);
      strokeWeight(3);
      line(x1, y1, -3, x2, y2, -3);
      line(x1, y1,  3, x2, y2,  3);

      if (s % 2 == 0) {
        stroke(240, 240, 235);
        strokeWeight(2.5);
        line(x1, y1, -90, x2, y2, -90);
        line(x1, y1,  90, x2, y2,  90);
      }
    }

    // Approach Road Streetlights continuing onto both shores
    for (int k = 0; k < 5; k++) {
      float ax = 1060 + k * 140;
      float lx = side * ax;
      float ly = getBridgeRoadSurfaceY(lx);

      for (int zSide = -1; zSide <= 1; zSide += 2) {
        pushMatrix();
        translate(lx, ly, zSide * 155);
        noStroke();
        fill(65, 65, 60);
        pushMatrix();
        translate(0, -67.5, 0);
        box(5, 135, 5);
        popMatrix();

        pushMatrix();
        translate(zSide * -17.5, -139, 0);
        rotateZ(zSide * 0.38);
        box(38, 4, 4);
        popMatrix();

        fill(255, 235, 170);
        pushMatrix();
        translate(zSide * -34, -146, 0);
        sphereDetail(7);
        sphere(8);
        popMatrix();
        popMatrix();
      }
    }
  }
}

// ============================================================
// ANIMATED VAIGAI RIVER
// ============================================================

void drawRiver() {
  noStroke();
  int nx = 70;
  int nz = 55;
  float dx = riverLength / nx;
  float activeW = getActiveWaterHalfWidth() * 2.0;
  float dz = activeW / nz;

  for (int iz = 0; iz < nz; iz++) {
    beginShape(TRIANGLE_STRIP);
    for (int ix = 0; ix <= nx; ix++) {
      float x = -riverLength / 2 + ix * dx;
      float z1 = -activeW / 2 + iz * dz;
      float z2 = z1 + dz;

      float w1 = waterWave(x, z1);
      float w2 = waterWave(x, z2);

      fill(80 + w1, 105 + w1 * 0.7, 112 + w1 * 0.5);
      vertex(x, waterLevel + w1, z1);

      fill(80 + w2, 105 + w2 * 0.7, 112 + w2 * 0.5);
      vertex(x, waterLevel + w2, z2);
    }
    endShape();
  }
}

float waterWave(float x, float z) {
  float turbBoost = (flowRegime == 2) ? 1.8 : ((flowRegime == 1) ? 1.0 : 0.4);
  float a = sin(x * 0.035 - sceneTime * 3.2);
  float b = cos(z * 0.045 - sceneTime * 1.5);
  float c = sin((x + z) * 0.018 - sceneTime * 2.0);
  return (a * 2.5 + b * 2.0 + c * 1.5) * turbBoost;
}

// ============================================================
// HYDRODYNAMIC VELOCITY FIELD
// Aligned in world coordinates with the central river piers
// ============================================================

PVector getRiverVelocity(float x, float z, float t) {
  float activeHalfZ = getActiveWaterHalfWidth();
  float normZ = constrain(z / activeHalfZ, -1.0, 1.0);
  float profile = 1.0 - 0.62 * normZ * normZ;
  float vx = 4.4 * profile;
  float vz = 0.0;

  if (flowRegime == 0) {
    vz = 0.25 * sin(x * 0.007) * cos(z * 0.009);
  } else if (flowRegime == 1) {
    vz = 0.52 * sin(x * 0.008 - t * 2.2) * cos(z * 0.009);
  } else {
    float eddy1 = sin(x * 0.016 - t * 4.8) * cos(z * 0.018 + t * 3.1);
    float eddy2 = cos(x * 0.031 + t * 5.6) * sin(z * 0.027 - t * 4.2);
    float eddy3 = sin((x - z) * 0.024 - t * 6.4);
    vx += 1.75 * eddy1;
    vz = 2.75 * eddy1 + 1.95 * eddy2 + 1.35 * eddy3;
  }

  // Venturi constriction acceleration through the central parabolic arch openings
  float blockageRatio = constrain(getPierWidth() / spanLength, 0.12, 0.45);
  float bridgeBand = exp(-(x * x) / (240.0 * 240.0));
  vx += bridgeBand * blockageRatio * 3.8 * profile;

  float pierW = getPierWidth();
  float radiusSq = max(900.0, (pierW * 0.72) * (pierW * 0.72));
  float[] ribZ = {-115.0, 0.0, 115.0};

  // Exact hydrodynamic interaction with the central river piers (excluding shore quay piers)
  for (int i = 1; i < archCount; i++) {
    float localPierX = -bridgeLength / 2 + i * spanLength;
    float pierWorldZ = -localPierX;

    for (int r = 0; r < ribZ.length; r++) {
      float pierWorldX = ribZ[r];
      float dx = x - pierWorldX;
      float dz = z - pierWorldZ;
      float distSq = dx * dx + dz * dz;

      if (distSq < radiusSq * 4.5 && distSq > 1.0) {
        float strength = radiusSq / distSq;
        vx -= 1.65 * strength * (dx * dx - dz * dz) / distSq;
        vz += 2.85 * strength * (dx * dz) / distSq;

        if (dx > 0 && dx < 280 && abs(dz) < pierW * 0.9) {
          float wakeDecay = exp(-dx * 0.011);
          vx -= 2.15 * wakeDecay * cos(dz * 0.04);
          if (flowRegime == 0) {
            vz += sin(dx * 0.05 + i) * 0.32 * wakeDecay;
          } else if (flowRegime == 1) {
            vz += sin(dx * 0.08 - t * 5.5 + i) * 1.15 * wakeDecay;
          } else {
            float wakeTurb = sin(dx * 0.11 - t * 9.5 + i * 1.7) + 0.7 * cos(dz * 0.14 + t * 8.0);
            vz += wakeTurb * 2.95 * wakeDecay;
          }
        }
      }
    }
  }

  vx = max(0.35, vx);
  return new PVector(vx, vz);
}

// ============================================================
// UPDATE FLOW VECTORS, STREAKLINES & PATHLINES
// ============================================================

void updateRiverFlow() {
  float waterZLimit = getActiveWaterHalfWidth() - 18.0;

  for (int i = 0; i < flowParticleCount; i++) {
    PVector vel = getRiverVelocity(flowPartX[i], flowPartZ[i], sceneTime);
    flowPartVel[i].set(vel.x, vel.y);
    flowPartMag[i] = vel.mag();

    flowPartX[i] += vel.x * 1.05;
    flowPartZ[i] += vel.y * 1.05;
    flowPartZ[i] = constrain(flowPartZ[i], -waterZLimit, waterZLimit);

    if (flowPartX[i] > riverLength / 2) {
      flowPartX[i] = -riverLength / 2;
      flowPartZ[i] = random(-waterZLimit + 10, waterZLimit - 10);
      PVector resetVel = getRiverVelocity(flowPartX[i], flowPartZ[i], sceneTime);
      flowPartVel[i].set(resetVel.x, resetVel.y);
      flowPartMag[i] = resetVel.mag();
    }
  }

  updateStreaklines(sceneTime);
  updatePathlines(sceneTime);
}

void updateStreaklines(float t) {
  float waterZLimit = getActiveWaterHalfWidth() - 18.0;

  for (int e = 0; e < streakEmitterCount; e++) {
    float zBase = map(e, 0, streakEmitterCount - 1, -waterZLimit + 35, waterZLimit - 35);
    emitterZ[e] = zBase;

    for (int k = streakTrailLength - 1; k > 0; k--) {
      PVector vel = getRiverVelocity(streakX[e][k - 1], streakZ[e][k - 1], t);
      streakX[e][k] = streakX[e][k - 1] + vel.x * 4.2;
      streakZ[e][k] = constrain(streakZ[e][k - 1] + vel.y * 4.2, -waterZLimit, waterZLimit);
    }

    streakX[e][0] = -riverLength / 2 + 70;
    streakZ[e][0] = emitterZ[e];
  }
}

void updatePathlines(float t) {
  float waterZLimit = getActiveWaterHalfWidth() - 18.0;

  for (int p = 0; p < pathParticleCount; p++) {
    int count = pathCount[p];
    float curX = pathX[p][count - 1];
    float curZ = pathZ[p][count - 1];

    PVector vel = getRiverVelocity(curX, curZ, t);
    float nextX = curX + vel.x * 3.8 * pathSpeedFactor[p];
    float nextZ = constrain(curZ + vel.y * 3.8 * pathSpeedFactor[p], -waterZLimit, waterZLimit);

    if (nextX > riverLength / 2 - 20) {
      resetPathParticle(p, false);
    } else {
      if (count < pathHistoryLength) {
        pathX[p][count] = nextX;
        pathZ[p][count] = nextZ;
        pathCount[p] = count + 1;
      } else {
        for (int k = 0; k < pathHistoryLength - 1; k++) {
          pathX[p][k] = pathX[p][k + 1];
          pathZ[p][k] = pathZ[p][k + 1];
        }
        pathX[p][pathHistoryLength - 1] = nextX;
        pathZ[p][pathHistoryLength - 1] = nextZ;
      }
    }
  }
}

// ============================================================
// DRAW STREAMLINES, STREAKLINES, PATHLINES & VELOCITY SPHERES
// ============================================================

void drawRiverHydrodynamics() {
  float surfaceOffset = -3.0;
  float waterZLimit = getActiveWaterHalfWidth() - 15.0;
  sphereDetail(7);

  // 1. STREAMLINES (Cyan)
  if (showStreamlines) {
    for (int s = 0; s < streamLineCount; s++) {
      float startZ = map(s, 0, streamLineCount - 1, -waterZLimit + 25, waterZLimit - 25);
      float curX = -riverLength / 2 + 25;
      float curZ = startZ;

      strokeWeight(1.8);
      noFill();
      beginShape();
      for (int step = 0; step < 95; step++) {
        float waveY = waterLevel + waterWave(curX, curZ) + surfaceOffset;
        float alpha = map(sin(PI * step / 95.0), 0, 1, 40, 185);
        stroke(0, 235, 255, alpha);
        vertex(curX, waveY, curZ);

        PVector vel = getRiverVelocity(curX, curZ, sceneTime);
        curX += vel.x * 6.6;
        curZ = constrain(curZ + vel.y * 6.6, -waterZLimit, waterZLimit);
        if (curX > riverLength / 2 - 20) break;
      }
      endShape();
    }
  }

  // 2. STREAKLINES (Magenta)
  if (showStreaklines) {
    for (int e = 0; e < streakEmitterCount; e++) {
      float injX = streakX[e][0];
      float injZ = constrain(streakZ[e][0], -waterZLimit, waterZLimit);
      float injY = waterLevel + waterWave(injX, injZ) + surfaceOffset - 2.5;
      pushMatrix();
      translate(injX, injY, injZ);
      noStroke();
      fill(255, 255, 255);
      sphere(6.0);
      popMatrix();

      strokeWeight(2.8);
      noFill();
      beginShape();
      for (int k = 0; k < streakTrailLength; k++) {
        float sx = streakX[e][k];
        float sz = constrain(streakZ[e][k], -waterZLimit, waterZLimit);
        if (sx > riverLength / 2 - 15) break;
        float sy = waterLevel + waterWave(sx, sz) + surfaceOffset - 1.5;
        float fade = map(k, 0, streakTrailLength - 1, 240, 50);
        stroke(255, 50, 180, fade);
        vertex(sx, sy, sz);
      }
      endShape();
    }
  }

  // 3. PATHLINES (Gold)
  if (showPathlines) {
    for (int p = 0; p < pathParticleCount; p++) {
      int count = pathCount[p];
      if (count < 2) continue;

      strokeWeight(2.6);
      noFill();
      beginShape();
      for (int k = 0; k < count; k++) {
        float px = pathX[p][k];
        float pz = constrain(pathZ[p][k], -waterZLimit, waterZLimit);
        float py = waterLevel + waterWave(px, pz) + surfaceOffset - 2.2;
        float alpha = map(k, 0, count - 1, 30, 235);
        stroke(255, 195, 0, alpha);
        vertex(px, py, pz);
      }
      endShape();
    }
  }

  // 4. VELOCITY PARTICLES & DIRECTION VECTORS
  if (showFlowlines) {
    for (int i = 0; i < flowParticleCount; i++) {
      float px = flowPartX[i];
      float pz = constrain(flowPartZ[i], -waterZLimit, waterZLimit);
      PVector vel = flowPartVel[i];
      float speed = flowPartMag[i];

      color particleColor = getVelocityColor(speed, 250);
      color vectorColor   = getVelocityColor(speed, 225);

      float py = waterLevel + waterWave(px, pz) + surfaceOffset - 2.2;

      float sphereRadius = map(constrain(speed, minSpeedScale, maxSpeedScale), minSpeedScale, maxSpeedScale, 3.8, 5.8);
      pushMatrix();
      translate(px, py, pz);
      noStroke();
      fill(particleColor);
      sphere(sphereRadius);
      popMatrix();

      if (speed > 0.05) {
        PVector dir = vel.copy().normalize();
        float vecLength = map(constrain(speed, minSpeedScale, maxSpeedScale), minSpeedScale, maxSpeedScale, 14.0, 56.0);

        float tipX = px + dir.x * vecLength;
        float tipZ = constrain(pz + dir.y * vecLength, -waterZLimit, waterZLimit);
        float tipY = waterLevel + waterWave(tipX, tipZ) + surfaceOffset - 2.2;

        stroke(vectorColor);
        strokeWeight(2.4);
        line(px, py, pz, tipX, tipY, tipZ);

        float angle = atan2(dir.y, dir.x);
        float headSize = map(constrain(speed, minSpeedScale, maxSpeedScale), minSpeedScale, maxSpeedScale, 6.0, 11.5);
        float ax1 = tipX - cos(angle - 0.42) * headSize;
        float az1 = tipZ - sin(angle - 0.42) * headSize;
        float ax2 = tipX - cos(angle + 0.42) * headSize;
        float az2 = tipZ - sin(angle + 0.42) * headSize;

        line(tipX, tipY, tipZ, ax1, tipY, az1);
        line(tipX, tipY, tipZ, ax2, tipY, az2);
      }
    }
  }
}

// ============================================================
// BRIDGE SUPPORTS, TRUE PARABOLIC ARCHES, DECK, RAILINGS & LIGHTS
// ============================================================

void drawBridgeSupports() {
  float pierW = getPierWidth();
  float[] ribZ = {-115.0, 0.0, 115.0};

  for (int i = 0; i <= archCount; i++) {
    float x = -bridgeLength / 2 + i * spanLength;
    boolean isEndPier = (i == 0 || i == 1 || i == archCount - 1 || i == archCount);

    for (int r = 0; r < ribZ.length; r++) {
      float zPos = ribZ[r];

      pushMatrix();
      translate(x, 207.5, zPos);
      noStroke();
      // Slightly warmer masonry tone on the last two end pillars framing the under-arch roads
      if (isEndPier) {
        fill(176, 122, 84);
      } else {
        fill(165, 115, 80);
      }
      box(pierW, 175, 36);

      pushMatrix();
      translate(0, 101.5, 0);
      fill(125, 125, 115);
      box(pierW + 16, 54, 52);
      popMatrix();
      popMatrix();
    }

    pushMatrix();
    translate(x, 124, 0);
    noStroke();
    fill(150, 100, 70);
    box(pierW + 6, 18, bridgeWidth - 36);
    popMatrix();
  }
}

void drawSeparatedParabolicArches() {
  float pierW = getPierWidth();
  float halfClearSpan = (spanLength - pierW) * 0.5 + 2.0;

  for (int i = 0; i < archCount; i++) {
    float xStart = -bridgeLength / 2 + i * spanLength;
    float xEnd = -bridgeLength / 2 + (i + 1) * spanLength;
    float xCenter = (xStart + xEnd) * 0.5;

    for (float zCenter : new float[]{-115.0, 0.0, 115.0}) {
      drawOneParabolicArchSpan(xCenter, halfClearSpan, zCenter);
    }
  }
}

// True Mathematical Parabolic Arch: y(u) = archSpringY - archRise * (1 - u^2), u in [-1, 1]
void drawOneParabolicArchSpan(float xCenter, float halfSpanX, float zCenter) {
  int segments = 44;
  float archDepth = 44;
  float spandrelDepth = 36;
  float deckSoffitY = roadY + 44;

  float zLeft = zCenter - archDepth * 0.5;
  float zRight = zCenter + archDepth * 0.5;
  float zSpLeft = zCenter - spandrelDepth * 0.5;
  float zSpRight = zCenter + spandrelDepth * 0.5;

  noStroke();

  // 1. Spandrel wall between parabolic extrados and bridge deck soffit
  fill(158, 106, 72);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yOuter = archSpringY - (archRise + archThickness) * parabola;
    vertex(currentX, deckSoffitY, zSpLeft);
    vertex(currentX, yOuter, zSpLeft);
  }
  endShape();

  fill(142, 92, 60);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yOuter = archSpringY - (archRise + archThickness) * parabola;
    vertex(currentX, deckSoffitY, zSpRight);
    vertex(currentX, yOuter, zSpRight);
  }
  endShape();

  // 2. Parabolic Arch Rib (Front Face)
  fill(182, 122, 80);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yInner = archSpringY - archRise * parabola;
    float yOuter = archSpringY - (archRise + archThickness) * parabola;
    vertex(currentX, yInner, zLeft);
    vertex(currentX, yOuter, zLeft);
  }
  endShape();

  // 3. Parabolic Arch Rib (Back Face)
  fill(160, 100, 64);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yInner = archSpringY - archRise * parabola;
    float yOuter = archSpringY - (archRise + archThickness) * parabola;
    vertex(currentX, yInner, zRight);
    vertex(currentX, yOuter, zRight);
  }
  endShape();

  // 4. Parabolic Intrados (Underside Barrel of the Arch)
  fill(102, 62, 42);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yInner = archSpringY - archRise * parabola;
    vertex(currentX, yInner, zLeft);
    vertex(currentX, yInner, zRight);
  }
  endShape();

  // 5. Parabolic Extrados Top Lip
  fill(200, 140, 94);
  beginShape(QUAD_STRIP);
  for (int j = 0; j <= segments; j++) {
    float u = -1.0 + 2.0 * j / float(segments);
    float parabola = 1.0 - u * u;
    float currentX = xCenter + u * halfSpanX;
    float yOuter = archSpringY - (archRise + archThickness) * parabola;
    vertex(currentX, yOuter, zLeft);
    vertex(currentX, yOuter, zRight);
  }
  endShape();
}

// ============================================================
// 3D PARABOLIC ARCH DISTANCE DIMENSION INDICATOR
// ============================================================

void drawBridgeArchDimensionCallout() {
  int midIdx = archCount / 2;
  float x1 = -bridgeLength / 2 + midIdx * spanLength;
  float x2 = -bridgeLength / 2 + (midIdx + 1) * spanLength;
  float zFace = bridgeWidth * 0.5 + 14;
  float yDim = 252;

  stroke(255, 85, 65);
  strokeWeight(3.0);
  line(x1, yDim - 22, zFace, x1, yDim + 18, zFace);
  line(x2, yDim - 22, zFace, x2, yDim + 18, zFace);

  strokeWeight(4.0);
  line(x1, yDim, zFace, x2, yDim, zFace);

  float arrowSize = min(14.0, spanLength * 0.16);
  line(x1, yDim, zFace, x1 + arrowSize, yDim - 7, zFace);
  line(x1, yDim, zFace, x1 + arrowSize, yDim + 7, zFace);
  line(x2, yDim, zFace, x2 - arrowSize, yDim - 7, zFace);
  line(x2, yDim, zFace, x2 - arrowSize, yDim + 7, zFace);

  float pierW = getPierWidth();
  float halfClearSpan = (spanLength - pierW) * 0.5;
  float xCenter = (x1 + x2) * 0.5;
  stroke(255, 220, 60);
  strokeWeight(3.2);
  noFill();
  beginShape();
  for (int j = 0; j <= 32; j++) {
    float u = -1.0 + 2.0 * j / 32.0;
    float px = xCenter + u * halfClearSpan;
    float py = archSpringY - archRise * (1.0 - u * u);
    vertex(px, py, 138);
  }
  endShape();

  // Highlight the two end parabolic arch curves framing the under-arch riverbank roads
  int[] endArchIndices = {0, archCount - 1};
  stroke(65, 245, 175, 210);
  strokeWeight(2.6);
  for (int idx : endArchIndices) {
    float ex1 = -bridgeLength / 2 + idx * spanLength;
    float ex2 = -bridgeLength / 2 + (idx + 1) * spanLength;
    float eCenter = (ex1 + ex2) * 0.5;
    beginShape();
    for (int j = 0; j <= 28; j++) {
      float u = -1.0 + 2.0 * j / 28.0;
      float px = eCenter + u * halfClearSpan;
      float py = archSpringY - archRise * (1.0 - u * u);
      vertex(px, py, 138);
    }
    endShape();
  }
}

void drawBridgeDeck() {
  float fullDeckLength = bridgeRampStart * 2.0;

  pushMatrix();
  translate(0, roadY + 22, 0);
  noStroke();

  fill(165, 120, 85);
  box(fullDeckLength, 44, bridgeWidth);

  pushMatrix();
  translate(0, -24, 0);
  fill(55, 55, 53);
  box(fullDeckLength - 10, 6, bridgeWidth - 35);
  popMatrix();

  stroke(240, 195, 55);
  strokeWeight(3);
  line(-fullDeckLength / 2 + 10, -27.5, -3, fullDeckLength / 2 - 10, -27.5, -3);
  line(-fullDeckLength / 2 + 10, -27.5, 3, fullDeckLength / 2 - 10, -27.5, 3);

  stroke(240, 240, 235);
  strokeWeight(2.5);
  int dashCount = int(fullDeckLength / 28);
  for (int i = 0; i < dashCount; i++) {
    float x = -fullDeckLength / 2 + 14 + i * 28;
    line(x, -27.5, -90, x + 14, -27.5, -90);
    line(x, -27.5, 90, x + 14, -27.5, 90);
  }

  popMatrix();
}

void drawBridgeRailings() {
  float fullDeckLength = bridgeRampStart * 2.0;

  for (int side = -1; side <= 1; side += 2) {
    float zOffset = side * (bridgeWidth / 2 - 10);

    pushMatrix();
    translate(0, roadY - 10, zOffset);
    noStroke();
    fill(210, 205, 190);
    box(fullDeckLength, 48, 20);
    fill(40, 100, 65);
    translate(0, -25, 0);
    box(fullDeckLength + 5, 8, 25);
    popMatrix();

    int postCount = int(fullDeckLength / 14);
    for (int i = 0; i < postCount; i++) {
      float x = -fullDeckLength / 2 + 7 + i * 14;
      pushMatrix();
      translate(x, roadY - 45, zOffset);
      fill(225, 220, 205);
      noStroke();
      box(10, 32, 5);
      popMatrix();
    }
  }
}

void drawStreetLights() {
  float fullDeckLength = bridgeRampStart * 2.0 - 40;
  int lightCount = 15;
  float step = fullDeckLength / (lightCount - 1);

  for (int i = 0; i < lightCount; i++) {
    float x = -fullDeckLength / 2 + i * step;
    for (int side = -1; side <= 1; side += 2) {
      pushMatrix();
      translate(x, 0, side * 155);
      noStroke();

      fill(65, 65, 60);
      pushMatrix();
      translate(0, roadY - 72.5, 0);
      box(5, 135, 5);
      popMatrix();

      pushMatrix();
      translate(side * -17.5, roadY - 144, 0);
      rotateZ(side * 0.38);
      box(38, 4, 4);
      popMatrix();

      fill(255, 235, 170);
      pushMatrix();
      translate(side * -34, roadY - 151, 0);
      sphereDetail(7);
      sphere(8);
      popMatrix();
      popMatrix();
    }
  }
}

// ============================================================
// VEHICLES (MAIN BRIDGE + APPROACH RAMPS + BANK UNDERPASS TRAFFIC)
// ============================================================

float getVehicleHalfLength(int type) {
  if (type == 0) return 38;
  if (type == 1) return 28;
  if (type == 2) return 58;
  if (type == 3) return 48;
  return 26;
}

float getVehicleHalfWidth(int type) {
  if (type == 0) return 24;
  if (type == 1) return 22;
  if (type == 2) return 31;
  if (type == 3) return 28;
  return 10;
}

void drawVehicles() {
  float halfTrack = approachEnd;
  float fullTrack = halfTrack * 2.0;

  for (int i = 0; i < vehicleCount; i++) {
    if (simulationRunning) {
      float targetSpeed = vehicleBaseSpeed[i];
      float myHalfLen = getVehicleHalfLength(vehicleType[i]);

      for (int j = 0; j < vehicleCount; j++) {
        if (i == j) continue;
        if (abs(vehicleLane[i] - vehicleLane[j]) < 15 && vehicleDirection[i] == vehicleDirection[j]) {
          float gap = (vehiclePosition[j] - vehiclePosition[i]) * vehicleDirection[i];
          if (gap < 0) gap += fullTrack;
          float minSafeGap = myHalfLen + getVehicleHalfLength(vehicleType[j]) + 45;
          if (gap < minSafeGap) {
            targetSpeed = min(targetSpeed, vehicleSpeed[j] * 0.85);
            if (gap < minSafeGap * 0.75) {
              targetSpeed = 0.2;
            }
          }
        }
      }

      vehicleSpeed[i] = lerp(vehicleSpeed[i], targetSpeed, 0.15);
      vehiclePosition[i] += vehicleSpeed[i] * vehicleDirection[i];

      if (vehicleDirection[i] == 1 && vehiclePosition[i] > halfTrack) {
        vehiclePosition[i] -= fullTrack;
      } else if (vehicleDirection[i] == -1 && vehiclePosition[i] < -halfTrack) {
        vehiclePosition[i] += fullTrack;
      }
    }

    float surfY = getBridgeRoadSurfaceY(vehiclePosition[i]);
    float pitchAng = getBridgeRoadPitch(vehiclePosition[i], vehicleDirection[i]);

    pushMatrix();
    translate(vehiclePosition[i], surfY, vehicleLane[i]);
    if (vehicleDirection[i] == 1) {
      rotateY(HALF_PI);
    } else {
      rotateY(-HALF_PI);
    }
    rotateX(pitchAng);

    // Record exact 2D screen coordinates of vehicle center for touch selection & speed labels
    vehScreenX[i] = screenX(0, -24, 0);
    vehScreenY[i] = screenY(0, -24, 0);
    vehVisible[i] = (vehScreenX[i] >= -80 && vehScreenX[i] <= width + 80 &&
                     vehScreenY[i] >= -80 && vehScreenY[i] <= height + 80);

    // Draw the 3D vehicle mesh
    drawVehicle(vehicleType[i], vehicleColors[i]);

    // Draw glowing 3D selection ring if this vehicle is currently inspected on the speedometer
    if (i == selectedVehicle) {
      drawSelectedVehicleBeacon(getVehicleHalfLength(vehicleType[i]));
    }

    // Draw 3D Velocity Vector (Magnitude + Direction) above vehicle
    float vecLen = map(constrain(vehicleSpeed[i], 0.2, 6.8), 0.2, 6.8, 30.0, 145.0);
    float vecY = -58.0;
    vecTipScreenX[i] = screenX(0, vecY, vecLen);
    vecTipScreenY[i] = screenY(0, vecY, vecLen);

    if (showVehicleVectors || i == selectedVehicle) {
      drawVehicleVelocityVector3D(vehicleSpeed[i], vecLen, vecY, i == selectedVehicle);
    }

    popMatrix();
  }
}

// ============================================================
// RIVERBANK UNDERPASS ROAD VEHICLES (THROUGH LAST PARABOLA ON BOTH SIDES)
// ============================================================

void drawBankRoadVehicles() {
  float bankRoadW = getBankRoadWidth();
  float vScale = getBankVehicleScale();
  float laneOffset = bankRoadW * 0.24;
  float fullBankTrack = bankRoadHalfLength * 2.0;

  for (int i = 0; i < bankVehicleCount; i++) {
    int side = bankVehSide[i];
    int dir = bankVehDir[i];
    float roadCenterX = getBankRoadCenterX(side);
    float vehX = roadCenterX + bankVehLaneSide[i] * laneOffset;

    if (simulationRunning) {
      float targetSpeed = bankVehBaseSpeed[i];
      float myHalfLen = getVehicleHalfLength(bankVehType[i]) * vScale;

      for (int j = 0; j < bankVehicleCount; j++) {
        if (i == j) continue;
        if (bankVehSide[i] == bankVehSide[j] && bankVehDir[i] == bankVehDir[j]) {
          float gap = (bankVehZ[j] - bankVehZ[i]) * dir;
          if (gap < 0) gap += fullBankTrack;
          float minSafeGap = myHalfLen + getVehicleHalfLength(bankVehType[j]) * vScale + 34.0;
          if (gap < minSafeGap) {
            targetSpeed = min(targetSpeed, bankVehSpeed[j] * 0.85);
            if (gap < minSafeGap * 0.72) {
              targetSpeed = 0.25;
            }
          }
        }
      }

      bankVehSpeed[i] = lerp(bankVehSpeed[i], targetSpeed, 0.16);
      bankVehZ[i] += bankVehSpeed[i] * dir;

      if (dir == 1 && bankVehZ[i] > bankRoadHalfLength) {
        bankVehZ[i] -= fullBankTrack;
      } else if (dir == -1 && bankVehZ[i] < -bankRoadHalfLength) {
        bankVehZ[i] += fullBankTrack;
      }
    }

    pushMatrix();
    // Positioned exactly between the last two pillars, moving straight out vertically along localZ
    translate(vehX, bankRoadSurfaceY, bankVehZ[i]);
    if (dir == -1) {
      rotateY(PI);
    }

    int globalId = BANK_VEH_ID_OFFSET + i;
    boolean isSel = (selectedVehicle == globalId);

    // Record 2D screen coordinates for touch selection and speed/vector badges
    bankVehScreenX[i] = screenX(0, -22 * vScale, 0);
    bankVehScreenY[i] = screenY(0, -22 * vScale, 0);
    bankVehVisible[i] = (bankVehScreenX[i] >= -80 && bankVehScreenX[i] <= width + 80 &&
                         bankVehScreenY[i] >= -80 && bankVehScreenY[i] <= height + 80);

    float vecLen = map(constrain(bankVehSpeed[i], 0.2, 6.8), 0.2, 6.8, 26.0, 125.0);
    float vecY = -52.0 * vScale;
    bankVecTipScreenX[i] = screenX(0, vecY, vecLen);
    bankVecTipScreenY[i] = screenY(0, vecY, vecLen);

    if (isSel) {
      drawSelectedVehicleBeacon(getVehicleHalfLength(bankVehType[i]) * vScale + 6.0);
    }

    if (showVehicleVectors || isSel) {
      drawVehicleVelocityVector3D(bankVehSpeed[i], vecLen, vecY, isSel);
    }

    scale(vScale);
    drawVehicle(bankVehType[i], bankVehColors[i]);
    popMatrix();
  }
}

// ============================================================
// HIGH-SPEED SPORTBIKE RACER (EXACT GAP CALCULATION & THREADING)
// ============================================================

void updateBikeRacer() {
  float halfTrack = approachEnd;
  float fullTrack = halfTrack * 2.0;
  float requiredTotalWidth = RACER_HALF_WIDTH * 2.0 + 3.0; // 12.0 bike + 3.0 margin = 15.0 u (1.50 m)

  // 1. Evaluate every candidate lateral corridor across Z in [-12, +148]
  // Compute exact inner-edge-to-inner-edge clear gap between adjacent vehicles
  float bestZ = racerTargetZ;
  float lowestPenalty = 1e9;
  int bestLeftVeh = -1;
  int bestRightVeh = -1;
  float bestLeftEdgeZ = 10.0;
  float bestRightEdgeZ = 156.0;
  float bestMidZ = 91.0;
  float bestClearGap = 146.0;

  for (float candZ = -12.0; candZ <= 148.0; candZ += 2.0) {
    float leftEdgeZ = 10.0;   // Default left bound: just right of double yellow centerline (Z=0)
    float rightEdgeZ = 156.0; // Default right bound: inner edge of right bridge barrier
    int candLeftVeh = -1;
    int candRightVeh = -1;
    float penalty = 0.0;

    // Inspect all vehicles in the active longitudinal envelope [-vehHalfLen - 38, +340]
    for (int j = 0; j < vehicleCount; j++) {
      float dx = vehiclePosition[j] - racerX;
      if (dx < -halfTrack) dx += fullTrack;
      if (dx >  halfTrack) dx -= fullTrack;

      float vehHalfLen = getVehicleHalfLength(vehicleType[j]);
      float vehHalfWid = getVehicleHalfWidth(vehicleType[j]);

      if (dx > -(vehHalfLen + 38.0) && dx < 340.0) {
        float vLeft = vehicleLane[j] - vehHalfWid;
        float vRight = vehicleLane[j] + vehHalfWid;

        if (vehicleLane[j] <= candZ) {
          if (vRight > leftEdgeZ) {
            leftEdgeZ = vRight;
            candLeftVeh = j;
          }
        } else {
          if (vLeft < rightEdgeZ) {
            rightEdgeZ = vLeft;
            candRightVeh = j;
          }
        }

        // Strongly penalize crossing into oncoming southbound lane when oncoming traffic is near
        if (vehicleDirection[j] == -1 && dx > -40.0 && dx < 420.0 && candZ < vRight + RACER_HALF_WIDTH + 10.0) {
          penalty += 8000.0;
        }
      }
    }

    float clearGap = rightEdgeZ - leftEdgeZ;
    float midZ = (leftEdgeZ + rightEdgeZ) * 0.5;

    // Hard physical constraint: gap width MUST exceed bike width + safety margin
    if (clearGap < requiredTotalWidth) {
      penalty += 50000.0 + (requiredTotalWidth - clearGap) * 4000.0;
    } else {
      penalty += abs(candZ - midZ) * 1.4;
      penalty += max(0.0, 32.0 - clearGap) * 2.2;
    }

    // Prefer staying on the northbound side of the yellow centerline (Z >= 16)
    if (candZ < 16.0) {
      penalty += (16.0 - candZ) * 25.0;
    }

    // Smooth steering continuity
    penalty += abs(candZ - racerZ) * 0.25;

    if (penalty < lowestPenalty) {
      lowestPenalty = penalty;
      bestZ = candZ;
      bestLeftVeh = candLeftVeh;
      bestRightVeh = candRightVeh;
      bestLeftEdgeZ = leftEdgeZ;
      bestRightEdgeZ = rightEdgeZ;
      bestMidZ = midZ;
      bestClearGap = clearGap;
    }
  }

  // 2. Lock onto the exact calculated midpoint of the flanking vehicles' gap
  racerLeftVehIdx = bestLeftVeh;
  racerRightVehIdx = bestRightVeh;
  racerGapLeftEdgeZ = bestLeftEdgeZ;
  racerGapRightEdgeZ = bestRightEdgeZ;
  racerCalculatedGapUnits = bestClearGap;
  racerCalculatedMidZ = bestMidZ;
  racerGapPassable = (bestClearGap >= requiredTotalWidth);
  racerGapActive = (bestLeftVeh != -1 || bestRightVeh != -1);

  if (bestLeftVeh != -1 && bestRightVeh != -1) {
    racerTargetZ = bestMidZ;
  } else {
    racerTargetZ = constrain(bestZ, bestLeftEdgeZ + RACER_HALF_WIDTH + 6.0, bestRightEdgeZ - RACER_HALF_WIDTH - 6.0);
  }

  // Compute longitudinal X-anchor of the gap throat between the flanking vehicles
  float sumX = 0;
  int weightCount = 0;
  if (bestLeftVeh != -1) {
    float dxL = vehiclePosition[bestLeftVeh] - racerX;
    if (dxL < -halfTrack) dxL += fullTrack;
    if (dxL >  halfTrack) dxL -= fullTrack;
    sumX += racerX + dxL;
    weightCount++;
  }
  if (bestRightVeh != -1) {
    float dxR = vehiclePosition[bestRightVeh] - racerX;
    if (dxR < -halfTrack) dxR += fullTrack;
    if (dxR >  halfTrack) dxR -= fullTrack;
    sumX += racerX + dxR;
    weightCount++;
  }
  racerGapDimX = (weightCount > 0) ? (sumX / float(weightCount)) : (racerX + 70.0);

  // 3. Smooth high-precision lateral steering onto the calculated midpoint
  float zError = racerTargetZ - racerZ;
  float desiredVz = constrain(zError * 0.22, -4.8, 4.8);
  racerVz = lerp(racerVz, desiredVz, 0.32);
  racerZ = constrain(racerZ + racerVz, -16.0, 150.0);

  // Update live left and right clearances from bike edges to adjacent vehicles
  racerLeftClearUnits = (racerZ - RACER_HALF_WIDTH) - racerGapLeftEdgeZ;
  racerRightClearUnits = racerGapRightEdgeZ - (racerZ + RACER_HALF_WIDTH);

  // 4. Dynamic Forward Speed + Collision Protection
  float desiredVx = racerBaseSpeed + 0.45 * sin(sceneTime * 1.4);
  for (int j = 0; j < vehicleCount; j++) {
    float dx = vehiclePosition[j] - racerX;
    if (dx < -halfTrack) dx += fullTrack;
    if (dx >  halfTrack) dx -= fullTrack;

    float minLongGap = getVehicleHalfLength(vehicleType[j]) + 32.0;
    float minLatGap  = getVehicleHalfWidth(vehicleType[j]) + RACER_HALF_WIDTH + 2.5;

    if (dx > 0 && dx < minLongGap + 24.0 && abs(racerZ - vehicleLane[j]) < minLatGap) {
      desiredVx = min(desiredVx, max(vehicleSpeed[j] + 0.8, 3.8));
    }
  }

  racerVx = lerp(racerVx, desiredVx, 0.20);
  racerX += racerVx;

  if (racerX > halfTrack) {
    racerX -= fullTrack;
    racerTrailCount = 0;
  }

  // 5. Compute resultant speed, yaw heading, and MotoGP lean angle
  racerSpeed = sqrt(racerVx * racerVx + racerVz * racerVz);
  racerYawAngle = atan2(racerVz, racerVx);
  float targetLean = constrain(-racerVz * 0.115, -0.52, 0.52);
  racerLeanAngle = lerp(racerLeanAngle, targetLean, 0.26);

  // 6. Record aerodynamic speed slipstream trail
  if (racerTrailCount < racerTrailLength) {
    racerTrailX[racerTrailCount] = racerX;
    racerTrailZ[racerTrailCount] = racerZ;
    racerTrailCount++;
  } else {
    for (int k = 0; k < racerTrailLength - 1; k++) {
      racerTrailX[k] = racerTrailX[k + 1];
      racerTrailZ[k] = racerTrailZ[k + 1];
    }
    racerTrailX[racerTrailLength - 1] = racerX;
    racerTrailZ[racerTrailLength - 1] = racerZ;
  }
}

void drawBikeRacer() {
  if (simulationRunning) {
    updateBikeRacer();
  }

  // Draw glowing aerodynamic slipstream trail following the exact bridge & ramp profile
  if (racerTrailCount > 1) {
    noFill();
    strokeWeight(3.2);
    beginShape();
    for (int k = 0; k < racerTrailCount; k++) {
      float alpha = map(k, 0, racerTrailCount - 1, 15, 195);
      stroke(255, 85, 35, alpha);
      float trailY = getBridgeRoadSurfaceY(racerTrailX[k]) - 6.0;
      vertex(racerTrailX[k], trailY, racerTrailZ[k]);
    }
    endShape();
  }

  float surfY = getBridgeRoadSurfaceY(racerX);
  float pitchAng = getBridgeRoadPitch(racerX, 1);

  pushMatrix();
  translate(racerX, surfY, racerZ);

  // Align bike heading directly along its instantaneous velocity vector (racerVx, racerVz)
  rotateY(HALF_PI + racerYawAngle);
  rotateX(pitchAng);

  // Record 2D screen coordinates for touch selection & speed labels
  racerScreenX = screenX(0, -24, 0);
  racerScreenY = screenY(0, -24, 0);
  racerVisible = (racerScreenX >= -80 && racerScreenX <= width + 80 &&
                  racerScreenY >= -80 && racerScreenY <= height + 80);

  // Selection beacon ring if inspected on speedometer
  if (selectedVehicle == RACER_ID) {
    drawSelectedVehicleBeacon(34);
  }

  // Draw 3D Velocity Vector (Magnitude + Resultant Direction) above the racer
  float vecLen = map(constrain(racerSpeed, 0.2, 7.2), 0.2, 7.2, 30.0, 158.0);
  float vecY = -58.0;
  racerVecTipScreenX = screenX(0, vecY, vecLen);
  racerVecTipScreenY = screenY(0, vecY, vecLen);

  if (showVehicleVectors || selectedVehicle == RACER_ID) {
    drawVehicleVelocityVector3D(racerSpeed, vecLen, vecY, selectedVehicle == RACER_ID);
  }

  // Apply realistic cornering/weaving body lean around tire contact patch
  rotateZ(racerLeanAngle);
  drawSportbikeMesh();

  popMatrix();
}

// ============================================================
// 3D INTER-VEHICLE CAD DIMENSION LINES & GAP THREADING GUIDE
// ============================================================

void drawVehicleDimensionLines() {
  gapDimVisible = false;
  subGapVisible = false;
  trafficDimCount = 0;

  if (!showGapDimensions) return;

  float halfTrack = approachEnd;
  float fullTrack = halfTrack * 2.0;

  // ------------------------------------------------------------
  // 1. PRIMARY LATERAL DIMENSION LINE BETWEEN FLANKING VEHICLES
  //    (Where the Sportbike Racer calculates the gap & threads through)
  // ------------------------------------------------------------
  if (racerGapActive && (racerLeftVehIdx != -1 || racerRightVehIdx != -1)) {
    float z1 = racerGapLeftEdgeZ;
    float z2 = racerGapRightEdgeZ;
    float zMid = racerCalculatedMidZ;
    float xDim = racerGapDimX;
    float localSurfY = getBridgeRoadSurfaceY(xDim);
    float yDim = localSurfY - 36;

    color dimCol = racerGapPassable ? color(35, 255, 175, 245) : color(255, 80, 65, 245);

    // A) Vertical CAD Extension Lines dropping from dimension bar to exact vehicle inner edges
    stroke(dimCol);
    strokeWeight(2.2);
    line(xDim, yDim - 8, z1, xDim, localSurfY - 2, z1);
    line(xDim, yDim - 8, z2, xDim, localSurfY - 2, z2);

    // Highlight longitudinal edge rails along the flanking vehicles' inner sides
    if (racerLeftVehIdx != -1) {
      float lx = vehiclePosition[racerLeftVehIdx];
      float lHalf = getVehicleHalfLength(vehicleType[racerLeftVehIdx]);
      float ly = getBridgeRoadSurfaceY(lx) - 3;
      line(lx - lHalf, ly, z1, lx + lHalf, ly, z1);
    }
    if (racerRightVehIdx != -1) {
      float rx = vehiclePosition[racerRightVehIdx];
      float rHalf = getVehicleHalfLength(vehicleType[racerRightVehIdx]);
      float ry = getBridgeRoadSurfaceY(rx) - 3;
      line(rx - rHalf, ry, z2, rx + rHalf, ry, z2);
    }

    // B) Main Horizontal Dimension Line spanning between the two vehicles
    strokeWeight(3.4);
    line(xDim, yDim, z1, xDim, yDim, z2);

    // Double-headed CAD arrows at z1 and z2
    float arr = min(7.5, abs(z2 - z1) * 0.25);
    line(xDim, yDim, z1, xDim - arr * 0.65, yDim, z1 + arr);
    line(xDim, yDim, z1, xDim + arr * 0.65, yDim, z1 + arr);
    line(xDim, yDim, z2, xDim - arr * 0.65, yDim, z2 - arr);
    line(xDim, yDim, z2, xDim + arr * 0.65, yDim, z2 - arr);

    // C) Calculated Gap Midpoint Target Crosshair & Threading Trajectory Line
    noStroke();
    fill(255, 235, 65, 250);
    pushMatrix();
    translate(xDim, yDim, zMid);
    sphereDetail(8);
    sphere(4.5);
    popMatrix();

    // Dashed trajectory line from the racer right through the center of the dimension line
    stroke(255, 235, 65, 220);
    strokeWeight(2.6);
    float guideStartX = racerX + 26;
    float guideEndX = xDim + 85;
    if (guideEndX > guideStartX) {
      int segs = 12;
      for (int s = 0; s < segs; s += 2) {
        float t1 = float(s) / segs;
        float t2 = float(s + 1) / segs;
        float gx1 = lerp(guideStartX, guideEndX, t1);
        float gx2 = lerp(guideStartX, guideEndX, t2);
        float gz1 = lerp(racerZ, zMid, min(1.0, t1 * 1.35));
        float gz2 = lerp(racerZ, zMid, min(1.0, t2 * 1.35));
        line(gx1, getBridgeRoadSurfaceY(gx1) - 5, gz1, gx2, getBridgeRoadSurfaceY(gx2) - 5, gz2);
      }
    }

    // Record 2D screen coordinates for the main gap calculation callout badge
    gapDimScreenX = screenX(xDim, yDim - 14, zMid);
    gapDimScreenY = screenY(xDim, yDim - 14, zMid);
    gapDimVisible = (gapDimScreenX >= -80 && gapDimScreenX <= width + 80 &&
                     gapDimScreenY >= -80 && gapDimScreenY <= height + 80);

    // D) Live Left & Right Side-Clearance Sub-Dimensions when racer is entering/threading the gap
    float longOffset = abs(racerX - xDim);
    if (longOffset < 135 && racerZ > z1 && racerZ < z2) {
      float bikeLeftZ = racerZ - RACER_HALF_WIDTH;
      float bikeRightZ = racerZ + RACER_HALF_WIDTH;
      float ySub = getBridgeRoadSurfaceY(racerX) - 12;

      stroke(255, 215, 65, 235);
      strokeWeight(2.2);
      if (bikeLeftZ > z1 + 1.0) {
        line(racerX, ySub, z1, racerX, ySub, bikeLeftZ);
        line(racerX, ySub - 4, bikeLeftZ, racerX, ySub + 4, bikeLeftZ);
      }
      if (z2 > bikeRightZ + 1.0) {
        line(racerX, ySub, bikeRightZ, racerX, ySub, z2);
        line(racerX, ySub - 4, bikeRightZ, racerX, ySub + 4, bikeRightZ);
      }

      subGapScreenX = screenX(racerX, ySub - 12, racerZ);
      subGapScreenY = screenY(racerX, ySub - 12, racerZ);
      subGapVisible = (subGapScreenX >= -80 && subGapScreenX <= width + 80 &&
                       subGapScreenY >= -80 && subGapScreenY <= height + 80);
    }
  }

  // ------------------------------------------------------------
  // 2. LONGITUDINAL & LATERAL DIMENSION LINES BETWEEN TRAFFIC VEHICLES
  // ------------------------------------------------------------
  for (int i = 0; i < vehicleCount; i++) {
    if (trafficDimCount >= MAX_TRAFFIC_DIMS) break;

    float xi = vehiclePosition[i];
    float zi = vehicleLane[i];
    int diri = vehicleDirection[i];
    float halfLenI = getVehicleHalfLength(vehicleType[i]);
    float halfWidI = getVehicleHalfWidth(vehicleType[i]);

    // Find closest vehicle ahead in the same lane -> Bumper-to-bumper longitudinal dimension
    int closestAhead = -1;
    float minLongGap = 1e9;

    for (int j = 0; j < vehicleCount; j++) {
      if (i == j) continue;
      if (abs(zi - vehicleLane[j]) < 10 && diri == vehicleDirection[j]) {
        float centerDist = (vehiclePosition[j] - xi) * diri;
        if (centerDist < 0) centerDist += fullTrack;
        float clearBumperGap = centerDist - halfLenI - getVehicleHalfLength(vehicleType[j]);
        if (clearBumperGap > 18.0 && clearBumperGap < minLongGap) {
          minLongGap = clearBumperGap;
          closestAhead = j;
        }
      }
    }

    if (closestAhead != -1 && minLongGap < 185.0) {
      float xFrontI = xi + diri * halfLenI;
      float xRearJ  = xFrontI + diri * minLongGap;
      if (abs(xFrontI) < bridgeRampStart && abs(xRearJ) < bridgeRampStart) {
        float surfY = getBridgeRoadSurfaceY((xFrontI + xRearJ) * 0.5);
        float yDim = surfY - 12;
        float zOff = zi + diri * (halfWidI * 0.45);

        stroke(135, 215, 255, 175);
        strokeWeight(1.6);
        line(xFrontI, surfY - 2, zOff, xFrontI, yDim - 5, zOff);
        line(xRearJ,  surfY - 2, zOff, xRearJ,  yDim - 5, zOff);
        line(xFrontI, yDim, zOff, xRearJ, yDim, zOff);

        float a = 5.0 * diri;
        line(xFrontI, yDim, zOff, xFrontI + a, yDim, zOff - 3.2);
        line(xFrontI, yDim, zOff, xFrontI + a, yDim, zOff + 3.2);
        line(xRearJ,  yDim, zOff, xRearJ  - a, yDim, zOff - 3.2);
        line(xRearJ,  yDim, zOff, xRearJ  - a, yDim, zOff + 3.2);

        float midX = (xFrontI + xRearJ) * 0.5;
        float sx = screenX(midX, yDim - 6, zOff);
        float sy = screenY(midX, yDim - 6, zOff);
        if (sx >= 20 && sx <= width - 20 && sy >= 20 && sy <= height - 20) {
          trafficDimScreenX[trafficDimCount] = sx;
          trafficDimScreenY[trafficDimCount] = sy;
          trafficDimMeters[trafficDimCount] = minLongGap * UNIT_TO_METERS;
          trafficDimIsLateral[trafficDimCount] = false;
          trafficDimCount++;
        }
      }
    }

    // Lateral gap dimension line between inner lane (|Z|=54) and outer lane (|Z|=128) vehicles
    if (trafficDimCount < MAX_TRAFFIC_DIMS && abs(abs(zi) - 54.0) < 8.0 && abs(xi) < bridgeRampStart) {
      float targetOuterZ = (zi > 0) ? 128.0 : -128.0;
      for (int j = 0; j < vehicleCount; j++) {
        if (abs(vehicleLane[j] - targetOuterZ) < 8.0) {
          float dxPair = abs(vehiclePosition[j] - xi);
          if (dxPair < 45.0) {
            float zInner = (zi > 0) ? (zi + halfWidI) : (zi - halfWidI);
            float zOuter = (zi > 0) ? (vehicleLane[j] - getVehicleHalfWidth(vehicleType[j]))
                                    : (vehicleLane[j] + getVehicleHalfWidth(vehicleType[j]));
            float latClear = abs(zOuter - zInner);
            float midX = (xi + vehiclePosition[j]) * 0.5;
            float surfY = getBridgeRoadSurfaceY(midX);
            float yDim = surfY - 22;

            stroke(255, 200, 95, 195);
            strokeWeight(1.8);
            line(midX, surfY - 2, zInner, midX, yDim - 5, zInner);
            line(midX, surfY - 2, zOuter, midX, yDim - 5, zOuter);
            line(midX, yDim, zInner, midX, yDim, zOuter);

            float sgn = (zOuter > zInner) ? 1.0 : -1.0;
            line(midX, yDim, zInner, midX - 3.2, yDim, zInner + 4.5 * sgn);
            line(midX, yDim, zInner, midX + 3.2, yDim, zInner + 4.5 * sgn);
            line(midX, yDim, zOuter, midX - 3.2, yDim, zOuter - 4.5 * sgn);
            line(midX, yDim, zOuter, midX + 3.2, yDim, zOuter - 4.5 * sgn);

            float zMidPair = (zInner + zOuter) * 0.5;
            float sx = screenX(midX, yDim - 6, zMidPair);
            float sy = screenY(midX, yDim - 6, zMidPair);
            if (sx >= 20 && sx <= width - 20 && sy >= 20 && sy <= height - 20) {
              trafficDimScreenX[trafficDimCount] = sx;
              trafficDimScreenY[trafficDimCount] = sy;
              trafficDimMeters[trafficDimCount] = latClear * UNIT_TO_METERS;
              trafficDimIsLateral[trafficDimCount] = true;
              trafficDimCount++;
            }
            break;
          }
        }
      }
    }
  }
}

void drawSportbikeMesh() {
  noStroke();

  // Rear & Front Racing Slick Tires
  fill(22, 22, 24);
  pushMatrix();
  translate(0, -9, -19);
  rotateZ(HALF_PI);
  cylinder(9, 8);
  popMatrix();

  pushMatrix();
  translate(0, -9, 20);
  rotateZ(HALF_PI);
  cylinder(9, 7);
  popMatrix();

  // Brake Discs & Swingarm
  fill(175, 180, 190);
  pushMatrix();
  translate(0, -9, -19);
  rotateZ(HALF_PI);
  cylinder(5.5, 9);
  popMatrix();

  pushMatrix();
  translate(0, -9, 20);
  rotateZ(HALF_PI);
  cylinder(5.5, 8);
  popMatrix();

  // Front Forks
  stroke(215, 185, 65);
  strokeWeight(3);
  line(-3.5, -9, 20, -3.5, -24, 11);
  line( 3.5, -9, 20,  3.5, -24, 11);
  noStroke();

  // Lower Belly Pan (Carbon Black)
  fill(30, 32, 36);
  pushMatrix();
  translate(0, -13, 1);
  box(12, 8, 28);
  popMatrix();

  // Main Aerodynamic Fairing & Fuel Tank (Vibrant Racing Red-Orange)
  fill(255, 55, 25);
  pushMatrix();
  translate(0, -21, 5);
  box(13, 12, 26);
  popMatrix();

  // White Racing Stripe Accents
  fill(245, 248, 255);
  pushMatrix();
  translate(0, -22, 6);
  box(13.6, 4, 22);
  popMatrix();

  // Upswept Tail Cowl
  fill(255, 55, 25);
  pushMatrix();
  translate(0, -25, -15);
  rotateX(-0.28);
  box(10, 7, 16);
  popMatrix();

  // Tinted Aerodynamic Windscreen
  fill(60, 195, 235, 185);
  pushMatrix();
  translate(0, -29, 13);
  rotateX(0.55);
  box(11, 9, 2);
  popMatrix();

  // Twin Titanium Exhaust Pipes
  fill(185, 190, 200);
  pushMatrix();
  translate(-5.5, -16, -16);
  rotateX(-0.22);
  box(2.8, 2.8, 16);
  popMatrix();

  // Tucked-in Sportbike Rider (Racing Leathers)
  fill(28, 32, 42);
  pushMatrix();
  translate(0, -31, -2);
  rotateX(0.65);
  box(12, 18, 11);
  popMatrix();

  // Rider Racing Back Hump (Neon Yellow/Red)
  fill(255, 215, 35);
  pushMatrix();
  translate(0, -34, -4);
  rotateX(0.65);
  box(7.5, 11, 5);
  popMatrix();

  // Full-Face Aerodynamic Racing Helmet
  fill(245, 248, 255);
  pushMatrix();
  translate(0, -40, 7);
  sphereDetail(9);
  sphere(7.0);
  popMatrix();

  // Reflective Iridium Visor
  fill(20, 220, 255);
  pushMatrix();
  translate(0, -40, 12.2);
  box(8.8, 4.5, 4);
  popMatrix();

  // High-Intensity LED Headlight & Ground Beam Cone
  fill(220, 250, 255);
  pushMatrix();
  translate(0, -21, 18.5);
  sphereDetail(7);
  sphere(4.2);
  popMatrix();

  fill(190, 240, 255, 42);
  beginShape(TRIANGLES);
  vertex(0, -21, 19);
  vertex(-20, -4, 135);
  vertex( 20, -4, 135);
  endShape();

  // Bright LED Tail Light
  fill(255, 25, 25);
  pushMatrix();
  translate(0, -26, -23);
  box(7, 3.5, 2);
  popMatrix();
}

void drawSelectedVehicleBeacon(float halfLen) {
  float radius = halfLen + 18 + sin(frameCount * 0.18) * 4.0;
  noFill();
  stroke(255, 225, 55, 235);
  strokeWeight(3.5);
  beginShape();
  for (int a = 0; a <= 32; a++) {
    float ang = TWO_PI * a / 32.0;
    vertex(cos(ang) * radius, -4, sin(ang) * radius);
  }
  endShape();
}

void drawVehicleVelocityVector3D(float spd, float vecLen, float vecY, boolean isSelected) {
  color vecCol = isSelected ? color(255, 235, 65, 250) : getVehicleSpeedColor(spd, 235);

  noStroke();
  fill(vecCol);
  pushMatrix();
  translate(0, vecY, 0);
  sphereDetail(7);
  sphere(isSelected ? 5.5 : 4.2);
  popMatrix();

  stroke(vecCol);
  strokeWeight(isSelected ? 4.5 : 3.2);
  line(0, vecY, 0, 0, vecY, vecLen);

  float headLen = map(constrain(spd, 0.2, 7.2), 0.2, 7.2, 11.0, 24.0);
  float headWidth = headLen * 0.55;
  line(0, vecY, vecLen, -headWidth, vecY, vecLen - headLen);
  line(0, vecY, vecLen,  headWidth, vecY, vecLen - headLen);
  line(0, vecY, vecLen, 0, vecY - headWidth * 0.7, vecLen - headLen);
  line(0, vecY, vecLen, 0, vecY + headWidth * 0.7, vecLen - headLen);
}

void drawVehicle(int type, color bodyColor) {
  noStroke();

  if (type == 0) {
    fill(bodyColor);
    pushMatrix();
    translate(0, -18, 0);
    box(48, 25, 75);
    popMatrix();

    pushMatrix();
    translate(0, -35, -3);
    fill(95, 145, 165);
    box(38, 19, 38);
    popMatrix();

    drawWheels(22, -9, 24);
    drawVehicleLights(19, -18, 38);
  }

  if (type == 1) {
    fill(30, 35, 30);
    pushMatrix();
    translate(0, -19, 0);
    box(44, 25, 55);
    popMatrix();

    fill(235, 165, 20);
    pushMatrix();
    translate(0, -31, 0);
    box(42, 19, 53);
    popMatrix();

    fill(90, 145, 165);
    pushMatrix();
    translate(0, -38, 12);
    box(32, 13, 3);
    popMatrix();

    drawWheels(20, -8, 18);
    drawVehicleLights(17, -18, 28);
  }

  if (type == 2) {
    fill(bodyColor);
    pushMatrix();
    translate(0, -32, 0);
    box(62, 49, 115);
    popMatrix();

    for (int i = 0; i < 5; i++) {
      pushMatrix();
      translate(0, -41, -46 + i * 21);
      fill(100, 150, 170);
      box(63, 15, 3);
      popMatrix();
    }
    pushMatrix();
    translate(0, -45, 58);
    fill(80, 140, 160);
    box(50, 20, 3);
    popMatrix();

    drawWheels(30, -11, 42);
    drawVehicleLights(25, -18, 58);
  }

  if (type == 3) {
    fill(bodyColor);
    pushMatrix();
    translate(0, -28, 0);
    box(56, 42, 95);
    popMatrix();

    pushMatrix();
    translate(0, -38, 24);
    fill(95, 145, 165);
    box(46, 16, 3);
    popMatrix();

    drawWheels(27, -10, 34);
    drawVehicleLights(22, -18, 48);
  }

  if (type == 4) {
    stroke(35, 35, 35);
    strokeWeight(5);
    line(0, -10, -20, 0, -10, 20);
    noStroke();

    fill(25, 25, 25);
    pushMatrix();
    translate(0, -8, -19);
    rotateZ(HALF_PI);
    cylinder(8, 7);
    popMatrix();

    pushMatrix();
    translate(0, -8, 19);
    rotateZ(HALF_PI);
    cylinder(8, 7);
    popMatrix();

    fill(190, 35, 25);
    pushMatrix();
    translate(0, -20, 0);
    box(15, 14, 28);
    popMatrix();

    fill(45, 50, 55);
    pushMatrix();
    translate(0, -33, -2);
    box(12, 22, 12);
    popMatrix();

    fill(210, 190, 45);
    pushMatrix();
    translate(0, -49, 4);
    sphereDetail(7);
    sphere(8);
    popMatrix();

    noStroke();
    fill(255, 245, 180);
    pushMatrix();
    translate(0, -22, 24);
    sphere(4.5);
    popMatrix();

    fill(255, 35, 20);
    pushMatrix();
    translate(0, -19, -24);
    box(7, 5, 2);
    popMatrix();
  }
}

void drawVehicleLights(float halfWidth, float lampY, float frontZ) {
  noStroke();

  fill(255, 245, 190);
  pushMatrix();
  translate(-halfWidth, lampY, frontZ + 0.5);
  box(8, 6, 3);
  popMatrix();

  pushMatrix();
  translate(halfWidth, lampY, frontZ + 0.5);
  box(8, 6, 3);
  popMatrix();

  fill(255, 235, 150, 35);
  beginShape(TRIANGLES);
  vertex(-halfWidth - 3, lampY, frontZ + 2);
  vertex(-halfWidth - 26, lampY + 12, frontZ + 105);
  vertex(-halfWidth + 22, lampY + 12, frontZ + 105);

  vertex(halfWidth - 3, lampY, frontZ + 2);
  vertex(halfWidth - 22, lampY + 12, frontZ + 105);
  vertex(halfWidth + 26, lampY + 12, frontZ + 105);
  endShape();

  fill(255, 35, 20);
  pushMatrix();
  translate(-halfWidth, lampY, -frontZ - 0.5);
  box(7, 6, 3);
  popMatrix();

  pushMatrix();
  translate(halfWidth, lampY, -frontZ - 0.5);
  box(7, 6, 3);
  popMatrix();
}

void drawWheels(float x, float y, float z) {
  fill(25, 25, 25);
  float r = abs(y);
  for (int side = -1; side <= 1; side += 2) {
    for (int front = -1; front <= 1; front += 2) {
      pushMatrix();
      translate(side * x, y, front * z);
      rotateZ(HALF_PI);
      cylinder(r, 8);
      popMatrix();
    }
  }
}

void cylinder(float r, float h) {
  int sides = 12;
  beginShape(QUAD_STRIP);
  for (int i = 0; i <= sides; i++) {
    float a = TWO_PI * i / sides;
    float x = cos(a) * r;
    float z = sin(a) * r;
    vertex(x, -h / 2, z);
    vertex(x, h / 2, z);
  }
  endShape();
}

void drawWaterFoam() {
  strokeWeight(2);
  float activeHalfW = getActiveWaterHalfWidth() - 25.0;
  for (int i = 0; i < 120; i++) {
    float x = -riverLength / 2 + ((sceneTime * 95 + i * 113) % riverLength);
    float z = -activeHalfW + randomStable(i + 80) * (activeHalfW * 2.0);
    float y = waterLevel - 2.5;

    stroke(230, 235, 225, 85);
    line(x, y + waterWave(x, z), z, x + 20, y + waterWave(x + 20, z), z + 1.5);
  }
}

// ============================================================
// HUD, VEHICLE & RACER OVERLAYS, DIMENSION LABELS & SPEEDOMETER
// ============================================================

void drawInformation() {
  hint(DISABLE_DEPTH_TEST);
  camera();
  noLights();

  // 1. Floating 2D Inter-Vehicle Dimension Callout Labels
  if (showGapDimensions) {
    drawVehicleDimensionLabels();
  }

  // 2. Floating 2D Vehicle & Racer Speed + Vector Magnitude/Direction Labels
  drawVehicleTelemetryOverlays();

  // 3. Top Header & Fluid Velocity Scale Card
  float panelW = width - 32;
  noStroke();
  fill(15, 25, 35, 210);
  rect(16, 12, panelW, 112, 10);

  fill(255);
  textAlign(LEFT, TOP);
  textSize(18);
  text("VAIGAI RIVER BRIDGE - MADURAI", 28, 18);

  textSize(12);
  fill(220, 235, 245);
  String statusStr = simulationRunning ? "RUNNING" : "PAUSED";
  String regimeStr = (flowRegime == 0) 
    ? "STEADY FLOW" 
    : ((flowRegime == 1) 
    ? "LAMINAR FLOW" 
    : "TURBULENT FLOW");
  text("STATE: " + statusStr + "  |  REGIME: " + regimeStr + "  |  ARCHES: " + archCount, 28, 41);

  fill(255, 220, 110);
  text("ARCH: " + nf(spanLength, 1, 1) + " m  |  RISE: " + nf(archRise, 1, 1) + " m  |  TAP ANY VEHICLE OR RACER FOR SPEEDOMETER", 28, 59);

  // Velocity Color Legend Bar
  float barX = 28;
  float barY = 82;
  float barW = min(panelW - 24, 380);
  float barH = 10;
  int steps = 40;
  float stepW = barW / steps;
  for (int s = 0; s < steps; s++) {
    float spd = map(s, 0, steps - 1, minSpeedScale, maxSpeedScale);
    fill(getVelocityColor(spd, 255));
    rect(barX + s * stepW, barY, stepW + 1, barH);
  }

  textSize(10);
  fill(140, 195, 255);
  textAlign(LEFT, TOP);
  text("LOW |V| (BLUE)", barX, barY + 13);

  fill(255, 235, 90);
  textAlign(CENTER, TOP);
  text("MED (YELLOW)", barX + barW * 0.5, barY + 13);

  fill(255, 90, 75);
  textAlign(RIGHT, TOP);
  text("HIGH |V| (RED)", barX + barW, barY + 13);

  // 4. Draw 10 Interactive Toggle & Flow Regime Buttons
  textAlign(CENTER, CENTER);
  textSize(11);

  for (int i = 0; i < 10; i++) {
    boolean active = false;
    color accentColor = color(150);

    if (i == 0) { active = showStreamlines; accentColor = color(0, 235, 255); }
    if (i == 1) { active = showStreaklines; accentColor = color(255, 50, 180); }
    if (i == 2) { active = showPathlines; accentColor = color(255, 195, 0); }
    if (i == 3) { active = showFlowlines; accentColor = color(255, 145, 20); }
    if (i == 4) { active = (flowRegime == 0); accentColor = color(120, 210, 255); }
    if (i == 5) { active = (flowRegime == 1); accentColor = color(130, 245, 180); }
    if (i == 6) { active = (flowRegime == 2); accentColor = color(255, 110, 90); }
    if (i == 7) { active = showAllVehicleSpeeds; accentColor = color(95, 235, 145); }
    if (i == 8) { active = showVehicleVectors; accentColor = color(255, 210, 65); }
    if (i == 9) { active = showGapDimensions; accentColor = color(35, 255, 175); }

    strokeWeight(2);
    if (active) {
      stroke(accentColor);
      fill(20, 32, 44, 230);
    } else {
      stroke(100, 115, 130, 160);
      fill(20, 28, 36, 165);
    }
    rect(btnX[i], btnY[i], btnW[i], btnH[i], 8);

    if (i < 4 || i >= 7) {
      noStroke();
      fill(accentColor);
      ellipse(btnX[i] + 13, btnY[i] + btnH[i] / 2, 10, 10);
      fill(active ? 255 : 165);
      text(btnLabels[i], btnX[i] + btnW[i] / 2 + 7, btnY[i] + btnH[i] / 2);
    } else {
      fill(active ? accentColor : color(175));
      text(btnLabels[i], btnX[i] + btnW[i] / 2, btnY[i] + btnH[i] / 2);
    }
  }

  // 5. Draw Large Full-Width Smooth Sliders
  drawLargeArchSliders();

  // 6. Draw Interactive Speedometer Dial when any vehicle or the bike racer is touched
  boolean validTrafficSel = (selectedVehicle >= 0 && selectedVehicle < vehicleCount);
  boolean validRacerSel = (selectedVehicle == RACER_ID);
  boolean validBankSel = (selectedVehicle >= BANK_VEH_ID_OFFSET && selectedVehicle < BANK_VEH_ID_OFFSET + bankVehicleCount);
  if (validTrafficSel || validRacerSel || validBankSel) {
    drawSelectedVehicleSpeedometer(selectedVehicle);
  }

  hint(ENABLE_DEPTH_TEST);
}

void drawVehicleDimensionLabels() {
  float topHudLimit = sliderY[1] + sliderH[1] + 8;
  float bottomHudLimit = (selectedVehicle != -1) ? (dialCardY - 8) : (height - 12);

  // A) Subtle CAD spacing badges for traffic bumper-to-bumper & lane-to-lane dimension lines
  textAlign(CENTER, CENTER);
  textSize(9);
  for (int k = 0; k < trafficDimCount; k++) {
    float sx = trafficDimScreenX[k];
    float sy = trafficDimScreenY[k];
    if (sy < topHudLimit || sy > bottomHudLimit) continue;

    String tag = nf(trafficDimMeters[k], 1, 1) + "m";
    float tw = textWidth(tag) + 8;
    float th = 14;

    noStroke();
    fill(12, 22, 32, 185);
    rect(sx - tw * 0.5, sy - th * 0.5, tw, th, 4);

    fill(trafficDimIsLateral[k] ? color(255, 215, 115) : color(145, 225, 255));
    text(tag, sx, sy - 1);
  }

  // B) Primary Racer Gap-Calculation Callout Badge above the active lateral dimension line
  if (gapDimVisible && gapDimScreenY >= topHudLimit && gapDimScreenY <= bottomHudLimit) {
    float gapM = racerCalculatedGapUnits * UNIT_TO_METERS;
    float bikeM = RACER_HALF_WIDTH * 2.0 * UNIT_TO_METERS;
    String statusTag = racerGapPassable ? "OK" : "TIGHT";
    String gapText = "GAP: " + nf(gapM, 1, 2) + "m (BIKE " + nf(bikeM, 1, 2) + "m " + statusTag + ")";

    textSize(10);
    float gw = textWidth(gapText) + 14;
    float gh = 18;
    float gx = gapDimScreenX - gw * 0.5;
    float gy = gapDimScreenY - gh * 0.5 - 4;

    strokeWeight(1.8);
    stroke(racerGapPassable ? color(35, 255, 175) : color(255, 85, 65));
    fill(10, 26, 24, 232);
    rect(gx, gy, gw, gh, 5);

    noStroke();
    fill(racerGapPassable ? color(85, 255, 195) : color(255, 120, 105));
    textAlign(CENTER, CENTER);
    text(gapText, gapDimScreenX, gy + gh * 0.5 - 1);
  }

  // C) Live Left / Right Side-Clearance Readout while threading between vehicles
  if (subGapVisible && subGapScreenY >= topHudLimit && subGapScreenY <= bottomHudLimit) {
    float leftM = max(0.0, racerLeftClearUnits * UNIT_TO_METERS);
    float rightM = max(0.0, racerRightClearUnits * UNIT_TO_METERS);
    String subText = "L:" + nf(leftM, 1, 2) + "m | R:" + nf(rightM, 1, 2) + "m";

    textSize(9);
    float sw = textWidth(subText) + 10;
    float sh = 15;

    strokeWeight(1.3);
    stroke(255, 215, 65);
    fill(24, 20, 10, 225);
    rect(subGapScreenX - sw * 0.5, subGapScreenY - sh * 0.5, sw, sh, 4);

    noStroke();
    fill(255, 235, 110);
    textAlign(CENTER, CENTER);
    text(subText, subGapScreenX, subGapScreenY - 1);
  }
}

void drawVehicleTelemetryOverlays() {
  float topHudLimit = sliderY[1] + sliderH[1] + 8;
  float bottomHudLimit = (selectedVehicle != -1) ? (dialCardY - 8) : (height - 12);

  // 1. Regular bridge + approach road traffic vehicles
  for (int i = 0; i < vehicleCount; i++) {
    if (!vehVisible[i]) continue;

    float sx = vehScreenX[i];
    float sy = vehScreenY[i];
    if (sy < topHudLimit || sy > bottomHudLimit) continue;

    float kmh = vehicleSpeed[i] * speedToKmhScale;
    float mps = kmh / 3.6;
    color spdCol = getVehicleSpeedColor(vehicleSpeed[i], 245);
    boolean isSel = (i == selectedVehicle);

    if (showAllVehicleSpeeds || isSel) {
      String spdText = nf(kmh, 1, 0) + " km/h";
      textSize(isSel ? 12 : 10);
      float boxW = textWidth(spdText) + 12;
      float boxH = isSel ? 20 : 17;
      float bx = sx - boxW * 0.5;
      float by = sy - 34;

      strokeWeight(isSel ? 2.0 : 1.2);
      stroke(isSel ? color(255, 225, 65) : spdCol);
      fill(12, 20, 30, 215);
      rect(bx, by, boxW, boxH, 5);

      noStroke();
      fill(isSel ? color(255, 235, 95) : spdCol);
      textAlign(CENTER, CENTER);
      text(spdText, sx, by + boxH * 0.5 - 1);
    }

    if (showVehicleVectors || isSel) {
      float tx = vecTipScreenX[i];
      float ty = vecTipScreenY[i];
      if (ty >= topHudLimit && ty <= bottomHudLimit) {
        String dirTag = (vehicleDirection[i] == 1) ? "0° NB" : "180° SB";
        String vecText = nf(mps, 1, 1) + "m/s " + dirTag;

        textSize(isSel ? 11 : 9);
        float vw = textWidth(vecText) + 10;
        float vh = 15;

        noStroke();
        fill(15, 24, 34, 195);
        rect(tx - vw * 0.5, ty - vh - 4, vw, vh, 4);

        fill(isSel ? color(255, 235, 85) : color(235, 245, 255));
        textAlign(CENTER, CENTER);
        text(vecText, tx, ty - vh * 0.5 - 4);
      }
    }
  }

  // 2. Riverbank Underpass Road Vehicles (going straight out vertically through last parabola)
  for (int i = 0; i < bankVehicleCount; i++) {
    if (!bankVehVisible[i]) continue;

    float sx = bankVehScreenX[i];
    float sy = bankVehScreenY[i];
    if (sy < topHudLimit || sy > bottomHudLimit) continue;

    int globalId = BANK_VEH_ID_OFFSET + i;
    boolean isSel = (selectedVehicle == globalId);
    float kmh = bankVehSpeed[i] * speedToKmhScale;
    float mps = kmh / 3.6;
    color spdCol = getVehicleSpeedColor(bankVehSpeed[i], 245);

    if (showAllVehicleSpeeds || isSel) {
      String spdText = nf(kmh, 1, 0) + " km/h";
      textSize(isSel ? 11 : 9);
      float boxW = textWidth(spdText) + 10;
      float boxH = isSel ? 18 : 15;
      float bx = sx - boxW * 0.5;
      float by = sy - 28;

      strokeWeight(isSel ? 1.8 : 1.1);
      stroke(isSel ? color(255, 225, 65) : spdCol);
      fill(12, 24, 28, 210);
      rect(bx, by, boxW, boxH, 4);

      noStroke();
      fill(isSel ? color(255, 235, 95) : spdCol);
      textAlign(CENTER, CENTER);
      text(spdText, sx, by + boxH * 0.5 - 1);
    }

    if (showVehicleVectors || isSel) {
      float tx = bankVecTipScreenX[i];
      float ty = bankVecTipScreenY[i];
      if (ty >= topHudLimit && ty <= bottomHudLimit) {
        String dirTag = (bankVehDir[i] == 1) ? "90° EB" : "270° WB";
        String vecText = nf(mps, 1, 1) + "m/s " + dirTag;

        textSize(isSel ? 10 : 8);
        float vw = textWidth(vecText) + 8;
        float vh = 14;

        noStroke();
        fill(15, 24, 34, 190);
        rect(tx - vw * 0.5, ty - vh - 3, vw, vh, 4);

        fill(isSel ? color(255, 235, 85) : color(185, 250, 225));
        textAlign(CENTER, CENTER);
        text(vecText, tx, ty - vh * 0.5 - 3);
      }
    }
  }

  // 3. High-Speed Sportbike Racer Overlay Badge & Vector Readout
  if (racerVisible && racerScreenY >= topHudLimit && racerScreenY <= bottomHudLimit) {
    float rKmh = racerSpeed * speedToKmhScale;
    float rMps = rKmh / 3.6;
    boolean isRacerSel = (selectedVehicle == RACER_ID);

    if (showAllVehicleSpeeds || isRacerSel) {
      String rSpdText = "RACER " + nf(rKmh, 1, 0) + " km/h";
      textSize(isRacerSel ? 12 : 11);
      float boxW = textWidth(rSpdText) + 14;
      float boxH = 20;
      float bx = racerScreenX - boxW * 0.5;
      float by = racerScreenY - 38;

      strokeWeight(2.0);
      stroke(isRacerSel ? color(255, 235, 65) : color(255, 95, 45));
      fill(38, 14, 14, 230);
      rect(bx, by, boxW, boxH, 6);

      noStroke();
      fill(isRacerSel ? color(255, 240, 95) : color(255, 145, 95));
      textAlign(CENTER, CENTER);
      text(rSpdText, racerScreenX, by + boxH * 0.5 - 1);
    }

    if (showVehicleVectors || isRacerSel) {
      if (racerVecTipScreenY >= topHudLimit && racerVecTipScreenY <= bottomHudLimit) {
        int deg = round(degrees(racerYawAngle));
        String signStr = (deg > 0) ? "+" : "";
        String rVecText = nf(rMps, 1, 1) + "m/s " + signStr + deg + "°";

        textSize(10);
        float vw = textWidth(rVecText) + 10;
        float vh = 16;

        noStroke();
        fill(32, 16, 18, 215);
        rect(racerVecTipScreenX - vw * 0.5, racerVecTipScreenY - vh - 4, vw, vh, 4);

        fill(255, 225, 95);
        textAlign(CENTER, CENTER);
        text(rVecText, racerVecTipScreenX, racerVecTipScreenY - vh * 0.5 - 4);
      }
    }
  }
}

void drawSelectedVehicleSpeedometer(int idx) {
  boolean isRacer = (idx == RACER_ID);
  boolean isBankVeh = (idx >= BANK_VEH_ID_OFFSET);
  int bIdx = idx - BANK_VEH_ID_OFFSET;

  float rawSpd = isRacer ? racerSpeed : (isBankVeh ? bankVehSpeed[bIdx] : vehicleSpeed[idx]);
  float kmh = rawSpd * speedToKmhScale;
  float mps = kmh / 3.6;
  float maxScaleKmh = isRacer ? 160.0 : 100.0;
  float dialVal = constrain(displayedDialKmh, 0, maxScaleKmh);
  color spdCol = getVehicleSpeedColor(rawSpd, 255);

  // Main Speedometer HUD Card
  stroke(isRacer ? color(255, 105, 55, 240) : (isBankVeh ? color(65, 245, 175, 235) : color(255, 215, 70, 230)));
  strokeWeight(2.5);
  fill(14, 22, 32, 238);
  rect(dialCardX, dialCardY, dialCardW, dialCardH, 16);

  // Close [X] Button
  stroke(255, 110, 95);
  strokeWeight(2);
  fill(35, 22, 26, 230);
  ellipse(dialCloseX, dialCloseY, dialCloseR * 2, dialCloseR * 2);
  stroke(255, 220, 220);
  strokeWeight(2.2);
  line(dialCloseX - 5, dialCloseY - 5, dialCloseX + 5, dialCloseY + 5);
  line(dialCloseX + 5, dialCloseY - 5, dialCloseX - 5, dialCloseY + 5);

  // Analog Speedometer Dial Geometry (Left side of card)
  float cx = dialCardX + 94;
  float cy = dialCardY + 100;
  float dialR = 68;
  float startAng = radians(135);
  float endAng = radians(405);

  stroke(85, 105, 125);
  strokeWeight(3);
  fill(10, 16, 24);
  ellipse(cx, cy, dialR * 2 + 12, dialR * 2 + 12);

  noFill();
  strokeWeight(7);
  stroke(60, 215, 130, 190);
  arc(cx, cy, dialR * 1.72, dialR * 1.72, startAng, lerp(startAng, endAng, 0.45));
  stroke(255, 205, 45, 200);
  arc(cx, cy, dialR * 1.72, dialR * 1.72, lerp(startAng, endAng, 0.45), lerp(startAng, endAng, 0.75));
  stroke(255, 75, 55, 210);
  arc(cx, cy, dialR * 1.72, dialR * 1.72, lerp(startAng, endAng, 0.75), endAng);

  textAlign(CENTER, CENTER);
  textSize(9);
  int maxV = int(maxScaleKmh);
  int stepV = isRacer ? 20 : 10;
  int majorV = isRacer ? 40 : 20;

  for (int v = 0; v <= maxV; v += stepV) {
    float frac = v / maxScaleKmh;
    float ang = lerp(startAng, endAng, frac);
    boolean isMajor = (v % majorV == 0);
    float rOuter = dialR - 3;
    float rInner = isMajor ? (dialR - 13) : (dialR - 8);

    stroke(isMajor ? 245 : 145);
    strokeWeight(isMajor ? 2.0 : 1.2);
    line(cx + cos(ang) * rInner, cy + sin(ang) * rInner,
         cx + cos(ang) * rOuter, cy + sin(ang) * rOuter);

    if (isMajor) {
      float rText = dialR - 23;
      fill(220, 235, 245);
      noStroke();
      text(str(v), cx + cos(ang) * rText, cy + sin(ang) * rText);
    }
  }

  noStroke();
  fill(150, 175, 195);
  textSize(9);
  text("km/h", cx, cy + 28);

  float needleAng = lerp(startAng, endAng, dialVal / maxScaleKmh);
  stroke(255, 70, 55);
  strokeWeight(3.2);
  line(cx - cos(needleAng) * 10, cy - sin(needleAng) * 10,
       cx + cos(needleAng) * (dialR - 10), cy + sin(needleAng) * (dialR - 10));

  noStroke();
  fill(240, 245, 250);
  ellipse(cx, cy, 14, 14);
  fill(255, 75, 55);
  ellipse(cx, cy, 6, 6);

  // Digital Telemetry Panel (Right side of card)
  float infoX = dialCardX + 184;
  float infoY = dialCardY + 16;

  textAlign(LEFT, TOP);
  fill(isRacer ? color(255, 135, 85) : (isBankVeh ? color(85, 250, 185) : color(255, 220, 95)));
  textSize(13);
  String headerStr;
  if (isRacer) {
    headerStr = "BIKE RACER #1  •  SUPERBIKE";
  } else if (isBankVeh) {
    headerStr = "BANK ROAD #" + (bIdx + 1) + "  •  " + getVehicleTypeName(bankVehType[bIdx]);
  } else {
    headerStr = "VEHICLE #" + (idx + 1) + "  •  " + getVehicleTypeName(vehicleType[idx]);
  }
  text(headerStr, infoX, infoY);

  fill(spdCol);
  textSize(26);
  text(nf(kmh, 1, 1) + " km/h", infoX, infoY + 20);

  String dirDesc;
  String laneDesc;
  String gapDesc;
  String stateDesc;
  color stateCol;

  if (isRacer) {
    float yawDeg = degrees(racerYawAngle);
    String turnTag = (yawDeg > 1.5) ? "WEAVING RIGHT" : ((yawDeg < -1.5) ? "WEAVING LEFT" : "STRAIGHT LINE");
    float calcGapM = racerCalculatedGapUnits * UNIT_TO_METERS;
    float midZM = racerCalculatedMidZ;
    dirDesc = nf(yawDeg, 1, 1) + "° (" + turnTag + ")";
    laneDesc = "Z = " + nf(racerZ, 1, 1) + "m  ->  MID Z = " + nf(midZM, 1, 1) + "m";
    gapDesc = "CALC GAP   : " + nf(calcGapM, 1, 2) + " m  (BIKE 1.20 m)";
    stateDesc = racerGapPassable ? "GAP VERIFIED - THREADING" : "SEEKING WIDER GAP";
    stateCol = racerGapPassable ? color(65, 255, 180) : color(255, 145, 75);
  } else if (isBankVeh) {
    String sideName = (bankVehSide[bIdx] < 0) ? "SOUTH BANK ARCH" : "NORTH BANK ARCH";
    dirDesc = (bankVehDir[bIdx] == 1) ? "+Z UNDER-ARCH (90° EB)" : "-Z UNDER-ARCH (270° WB)";
    laneDesc = sideName + " (Z = " + int(bankVehZ[bIdx]) + " m)";
    gapDesc = "ARCH SPAN  : " + nf(getClearArchOpening(), 1, 1) + " m CLEAR";
    boolean braking = (bankVehSpeed[bIdx] < bankVehBaseSpeed[bIdx] * 0.75);
    stateDesc = braking ? "BRAKING (BANK TRAFFIC)" : "THREADING LAST PARABOLA";
    stateCol = braking ? color(255, 145, 85) : color(85, 250, 185);
  } else {
    dirDesc = (vehicleDirection[idx] == 1) ? "+X BRIDGE (0° NORTHBOUND)" : "-X BRIDGE (180° SOUTHBOUND)";
    laneDesc = "Z = " + int(vehicleLane[idx]) + " m";
    gapDesc = "WIDTH      : " + nf(getVehicleHalfWidth(vehicleType[idx]) * 2.0 * UNIT_TO_METERS, 1, 2) + " m";
    boolean braking = (vehicleSpeed[idx] < vehicleBaseSpeed[idx] * 0.75);
    stateDesc = braking ? "BRAKING (TRAFFIC GAP)" : "CRUISING STEADY";
    stateCol = braking ? color(255, 145, 85) : color(110, 240, 160);
  }

  fill(220, 235, 248);
  textSize(11);
  text("VECTOR |V| : " + nf(mps, 1, 2) + " m/s  (" + nf(rawSpd, 1, 2) + " u/f)", infoX, infoY + 56);
  text("DIRECTION  : " + dirDesc, infoX, infoY + 75);
  text("LANE POS   : " + laneDesc, infoX, infoY + 94);
  text(gapDesc, infoX, infoY + 113);

  fill(stateCol);
  text("STATUS     : " + stateDesc, infoX, infoY + 132);
}

void drawLargeArchSliders() {
  for (int s = 0; s < 2; s++) {
    float sx = sliderX[s];
    float sy = sliderY[s];
    float sw = sliderW[s];
    float sh = sliderH[s];
    boolean isSelected = (activeSlider == s);

    strokeWeight(isSelected ? 2.5 : 2.0);
    stroke(isSelected ? color(255, 210, 75) : color(95, 120, 145, 200));
    fill(15, 25, 36, 228);
    rect(sx, sy, sw, sh, 12);

    float norm = constrain(sliderNorm[s], 0.0, 1.0);

    String titleLabel = (s == 0)
      ? "PARABOLIC ARCH DISTANCE (PIER-TO-PIER SPAN)"
      : "PARABOLIC ARCH RISE HEIGHT";

    String valueLabel = (s == 0)
      ? nf(spanLength, 1, 1) + " m  (" + archCount + " Arches)"
      : nf(archRise, 1, 1) + " m";

    noStroke();
    textSize(12);
    fill(210, 225, 240);
    textAlign(LEFT, TOP);
    text(titleLabel, sx + 18, sy + 9);

    fill(s == 0 ? color(255, 130, 105) : color(255, 215, 85));
    textAlign(RIGHT, TOP);
    text(valueLabel, sx + sw - 18, sy + 9);

    float trackX1 = sx + 26;
    float trackX2 = sx + sw - 26;
    float trackY = sy + 45;
    float knobX = lerp(trackX1, trackX2, norm);

    if (s == 0) {
      stroke(110, 130, 150, 140);
      strokeWeight(2);
      for (int count = 6; count <= 20; count += 2) {
        float distVal = bridgeLength / float(count);
        float tNorm = constrain((distVal - minArchDistance) / (maxArchDistance - minArchDistance), 0.0, 1.0);
        float tx = lerp(trackX1, trackX2, tNorm);
        line(tx, trackY - 9, tx, trackY + 9);
      }
    }

    stroke(55, 70, 88);
    strokeWeight(10);
    line(trackX1, trackY, trackX2, trackY);

    color activeTrackCol = (s == 0) ? color(255, 95, 70) : color(255, 195, 45);
    stroke(activeTrackCol);
    strokeWeight(10);
    line(trackX1, trackY, knobX, trackY);

    noStroke();
    if (isSelected) {
      fill(red(activeTrackCol), green(activeTrackCol), blue(activeTrackCol), 75);
      ellipse(knobX, trackY, 40, 40);
    }
    fill(255);
    ellipse(knobX, trackY, isSelected ? 28 : 24, isSelected ? 28 : 24);

    fill(activeTrackCol);
    ellipse(knobX, trackY, 12, 12);
  }
}

void updateSliderFromMouse(int s, float mx) {
  float trackX1 = sliderX[s] + 26;
  float trackX2 = sliderX[s] + sliderW[s] - 26;
  float norm = constrain((mx - trackX1) / (trackX2 - trackX1), 0.0, 1.0);
  sliderNorm[s] = norm;

  if (s == 0) {
    float newDist = lerp(minArchDistance, maxArchDistance, norm);
    updateBridgeArchSpacing(newDist);
  } else if (s == 1) {
    archRise = lerp(minArchRise, maxArchRise, norm);
  }
}

// ============================================================
// TOUCH & MOUSE INTERACTION HANDLERS
// ============================================================

void mousePressed() {
  previousMouseX = mouseX;
  previousMouseY = mouseY;
  dragging = false;
  activeSlider = -1;

  for (int s = 0; s < 2; s++) {
    if (mouseX >= sliderX[s] - 14 && mouseX <= sliderX[s] + sliderW[s] + 14 &&
        mouseY >= sliderY[s] - 10 && mouseY <= sliderY[s] + sliderH[s] + 10) {
      activeSlider = s;
      updateSliderFromMouse(s, mouseX);
      return;
    }
  }
}

void mouseDragged() {
  if (activeSlider != -1) {
    updateSliderFromMouse(activeSlider, mouseX);
    return;
  }

  float dx = mouseX - previousMouseX;
  float dy = mouseY - previousMouseY;

  if (abs(dx) + abs(dy) > 3) {
    dragging = true;
  }

  cameraYaw += dx * 0.006;
  cameraPitch += dy * 0.004;
  cameraPitch = constrain(cameraPitch, 0.08, 1.25);

  previousMouseX = mouseX;
  previousMouseY = mouseY;
}

void mouseReleased() {
  if (activeSlider != -1) {
    activeSlider = -1;
    dragging = false;
    return;
  }

  if (!dragging) {
    // 1. Check Speedometer Dial close button or card tap when open
    if (selectedVehicle != -1) {
      if (dist(mouseX, mouseY, dialCloseX, dialCloseY) <= dialCloseR + 14) {
        selectedVehicle = -1;
        dragging = false;
        return;
      }
      if (mouseX >= dialCardX && mouseX <= dialCardX + dialCardW &&
          mouseY >= dialCardY && mouseY <= dialCardY + dialCardH) {
        dragging = false;
        return;
      }
    }

    // 2. Check all 10 HUD buttons
    for (int i = 0; i < 10; i++) {
      if (mouseX >= btnX[i] && mouseX <= btnX[i] + btnW[i] &&
          mouseY >= btnY[i] && mouseY <= btnY[i] + btnH[i]) {
        handleButtonPress(i);
        dragging = false;
        return;
      }
    }

    // 3. Check if user touched the Sportbike Racer, a bridge vehicle, or a riverbank road vehicle
    int closestVeh = -1;
    float minHitDist = 58.0;

    if (racerVisible) {
      float dr = dist(mouseX, mouseY, racerScreenX, racerScreenY);
      if (dr < 64.0) {
        minHitDist = dr;
        closestVeh = RACER_ID;
      }
    }

    for (int i = 0; i < vehicleCount; i++) {
      if (!vehVisible[i]) continue;
      float d = dist(mouseX, mouseY, vehScreenX[i], vehScreenY[i]);
      if (d < minHitDist) {
        minHitDist = d;
        closestVeh = i;
      }
    }

    for (int i = 0; i < bankVehicleCount; i++) {
      if (!bankVehVisible[i]) continue;
      float d = dist(mouseX, mouseY, bankVehScreenX[i], bankVehScreenY[i]);
      if (d < minHitDist) {
        minHitDist = d;
        closestVeh = BANK_VEH_ID_OFFSET + i;
      }
    }

    if (closestVeh != -1) {
      selectedVehicle = closestVeh;
      if (closestVeh == RACER_ID) {
        displayedDialKmh = racerSpeed * speedToKmhScale;
      } else if (closestVeh >= BANK_VEH_ID_OFFSET) {
        displayedDialKmh = bankVehSpeed[closestVeh - BANK_VEH_ID_OFFSET] * speedToKmhScale;
      } else {
        displayedDialKmh = vehicleSpeed[closestVeh] * speedToKmhScale;
      }
      dragging = false;
      return;
    }

    // 4. Otherwise tap empty 3D scene to Pause / Resume simulation
    simulationRunning = !simulationRunning;
  }
  dragging = false;
}

void handleButtonPress(int index) {
  if (index == 0) showStreamlines = !showStreamlines;
  if (index == 1) showStreaklines = !showStreaklines;
  if (index == 2) showPathlines = !showPathlines;
  if (index == 3) showFlowlines = !showFlowlines;

  if (index == 4) {
    flowRegime = 0;
    initializeRiverFlow();
  }
  if (index == 5) {
    flowRegime = 1;
    initializeRiverFlow();
  }
  if (index == 6) {
    flowRegime = 2;
    initializeRiverFlow();
  }

  if (index == 7) showAllVehicleSpeeds = !showAllVehicleSpeeds;
  if (index == 8) showVehicleVectors = !showVehicleVectors;
  if (index == 9) showGapDimensions = !showGapDimensions;
}
