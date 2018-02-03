*A simple scene implemented with Three.js several years ago during Computer Graphics course in university following J. Dirksen book "Learning Three.js". Uses old version of Three.js and other libs, I don't know if it is compatible with modern versions.*

A castle on an island and three moving boats. Speed can be changed via controls in the top right corner (it also depends on FPS).

![](https://i.imgur.com/8m5wQ58.png)

Can be run here: [https://alexp11223.github.io/Three.js_IslandCastle/index.html](https://alexp11223.github.io/Three.js_IslandCastle/index.html)

# Implementation details

## World

Lake/water is implemented using a 2D plane:

```javascript
var planeGeometry = new THREE.PlaneGeometry(1000, 500);
var planeMaterial = new THREE.MeshLambertMaterial({color: 0x1acffa}); 
var plane = new THREE.Mesh(planeGeometry, planeMaterial);
plane.receiveShadow  = true;
plane.rotation.x = -0.5*Math.PI;
scene.add(plane);
```

Background color is set to light blue to represent sky:

```javascript
renderer.setClearColor(0x9fd2f1, 1.0);
```

`HemisphereLight` + `DirectionalLight` are used to create a sun-like natural light.

`shadowBias` is set to `0.001` because I encountered a shadow artifact shown below and after some research ([https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx), [http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js](http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js)) I found out that it is called “**Peter Panning**” and can be fixed by adjusting `shadowBias` value.

![](https://i.imgur.com/AYlet59.png)

```javascript
var hemiLight = new THREE.HemisphereLight(0xffffff, 0xffffff, 0.6);
hemiLight.position.set(0, 500, 0);

scene.add(hemiLight);

var dirLight = new THREE.DirectionalLight(0xffffff, 0.5);
dirLight.castShadow = true;
dirLight.position.set(265, 150, -265);

dirLight.shadowMapWidth = 8192;
dirLight.shadowMapHeight = 8192;

var lightDist = 800;
dirLight.shadowCameraLeft = -lightDist;
dirLight.shadowCameraRight = lightDist;
dirLight.shadowCameraTop = lightDist;
dirLight.shadowCameraBottom = -lightDist;

dirLight.shadowCameraFar = 3500;
dirLight.shadowBias = 0.001;
dirLight.shadowDarkness = 0.35;

scene.add(dirLight);
```

The lake and the island receive shadows, all other objects (except some 2D objects such as doors and windows) cast shadows.

`TrackballControls.js` is used to allow changing camera position. Left mouse button and move — to rotate, right mouse button and move — to pan, scroll wheel — to zoom in/out:

```javascript
var cameraControls = new THREE.TrackballControls(camera, renderer.domElement);
cameraControls.rotateSpeed = 1.0;
cameraControls.zoomSpeed = 1.0;
cameraControls.panSpeed = 1.0;
cameraControls.noZoom = false;
cameraControls.noPan = false;
...
function render() {
    cameraControls.update();
```

## Boats

A boat consists of three parts:

1. A cube for the main part of the hull.

```javascript
var boatMaterial = new THREE.MeshLambertMaterial({color: boatColor});

var boat = new THREE.Mesh(new THREE.CubeGeometry(boatWidth, boatHeight, boatDepth), boatMaterial);
boat.castShadow = true;
```

2. A triangle for the sail.

Here also one more triangle with the same size and at almost the same position was added because if a boat is rotated to the opposite direction then a single 2D sail would not cast a shadow (even with `side` property).

```javascript
var sailGeometry = new THREE.Geometry();

sailGeometry.vertices.push(new THREE.Vector3(0, 0, 0));
sailGeometry.vertices.push(new THREE.Vector3(boatWidth + 3, 0, 0));
sailGeometry.vertices.push(new THREE.Vector3(boatWidth + 3, 36, 0));
// a 2D object (with material.side = DoubleSide) is not enough because shadow works only for one side
sailGeometry.vertices.push(new THREE.Vector3(0, 0, 0.1));
sailGeometry.vertices.push(new THREE.Vector3(boatWidth + 3, 0, 0.1));
sailGeometry.vertices.push(new THREE.Vector3(boatWidth + 3, 36, 0.1));

sailGeometry.faces.push(new THREE.Face3(0, 1, 2));
sailGeometry.faces.push(new THREE.Face3(5, 4, 3));
sailGeometry.computeFaceNormals();

var sailMaterial = new THREE.MeshLambertMaterial({color:0xffffff});

var sail = new THREE.Mesh(sailGeometry, sailMaterial);
sail.castShadow = true;
sail.position.set(-boatWidth / 2 - 8, boatHeight / 2, 0);
boat.add(sail);
```

3. A small 3D shape for the front part (*[bow](https://en.wikipedia.org/wiki/Bow_(ship))*): 4 vertices on the edges of the left side of the cube and another vertex further left (at the same height as the cube height) and 4 faces connecting them.

![https://i.imgur.com/Y4FFjYv.png](https://i.imgur.com/Y4FFjYv.png)

```javascript
var bowGeometry = new THREE.Geometry();
bowGeometry.vertices.push(new THREE.Vector3(0, boatHeight / 2, boatDepth / 2));
bowGeometry.vertices.push(new THREE.Vector3(0, -boatHeight / 2, boatDepth / 2));
bowGeometry.vertices.push(new THREE.Vector3(0, boatHeight / 2, -boatDepth / 2));
bowGeometry.vertices.push(new THREE.Vector3(0, -boatHeight / 2, -boatDepth / 2));
bowGeometry.vertices.push(new THREE.Vector3(-15, boatHeight / 2, 0));

bowGeometry.faces.push(new THREE.Face3(0, 2, 4));
bowGeometry.faces.push(new THREE.Face3(4, 1, 0));
bowGeometry.faces.push(new THREE.Face3(4, 3, 1));
bowGeometry.faces.push(new THREE.Face3(2, 3, 4));
bowGeometry.computeFaceNormals();

var bow = new THREE.Mesh(bowGeometry, boatMaterial);
bow.castShadow = true;
bow.position.set(-boatWidth / 2, 0, 0);
boat.add(bow);
```

3 boats with different colors and initial coordinates are created and added to the scene. Each boat has a speed stored in `step` property (negative if moving from right to left). In the rendering cycle each boat is moved by this value on the x axis, when a boat reaches some point far from the center it is rotated to the opposite direction (by changing `step` sign to the opposite and using `rotation.y` property to rotate the object itself).

```javascript
function rotateBoat(boat) {
    boat.rotation.y = boat.rotation.y === 0 ? Math.PI : 0; 
    boat.step *= -1;
}
...
boats.forEach(function(boat) {
    boat.step = -1;
    scene.add(boat);
});
rotateBoat(boats[2]);
...
function render() { 
    boats.forEach(function(boat) {
        boat.position.x += boat.step;

        if (Math.abs(boat.position.x) > 300) 
            rotateBoat(boat);
    });
```

Boats speed can be changed via dat.GUI control:

```javascript
var controls = new function() {
    this.windSpeed = 1;
};

var gui = new dat.GUI();
var speedControl = gui.add(controls, 'windSpeed', 0.05, 3);
speedControl.listen();
speedControl.onChange(function() {
    boats.forEach(function(boat) {
        boat.step = boat.step < 0 ? -controls.windSpeed : controls.windSpeed;
    });
});
```

## Island

The island is created using a green cylinder with top radius less than bottom radius.

```javascript
var island = new THREE.Mesh(new THREE.CylinderGeometry(103, 143, 10, 80), 
                             new THREE.MeshLambertMaterial({color: 0x00ff00}));
island.receiveShadow = true;
```

## Castle

All castle buildings use the same two materials for walls and roofs:

```javascript
var wallMaterial = new THREE.MeshLambertMaterial({color: 0x898989});
var roofMaterial = new THREE.MeshLambertMaterial({color: 0xff0000});
```

A castle wall is a cube with many small cubes on top of it (added in a loop along the x axis) representing battlements, and a tower on the right side:

```javascript
var wallGeometry = new THREE.CubeGeometry(wallWidth, wallHeight, wallDepth);
var wall = new THREE.Mesh(wallGeometry, wallMaterial);
wall.castShadow = true;
...

var battlementGeometry = new THREE.CubeGeometry(battlementSize, battlementSize, battlementSize);

for (var x = -(wallWidth / 2) + battlementSize / 2; x < wallWidth / 2; x += battlementSize * 2) {
    var battlement = new THREE.Mesh(battlementGeometry, wallMaterial);
    battlement.castShadow = true;
    battlement.position.set(x, wallHeight / 2 + battlementSize / 2, wallDepth / 2 - battlementSize / 2);
    wall.add(battlement);
}
var tower = createTower();
tower.position.x = wallWidth / 2;
wall.add(tower);
```

A tower is a cylinder with another cylinder on top of it (with top radius equal to 0) representing the roof.

```javascript
var tower = new THREE.Mesh(new THREE.CylinderGeometry(radius, radius, towerHeight, 20),  wallMaterial);
tower.castShadow = true;

var roof = new THREE.Mesh(new THREE.CylinderGeometry(0, radius, 16, 20), roofMaterial);
roof.castShadow = true;
roof.position.y = towerHeight - 15;
tower.add(roof);
```

4 walls are grouped into a rectangle and a gate added in the middle of the front wall.

```javascript
var castle = new THREE.Object3D();

var leftWall = createWall(wallWidth);
var rightWall = createWall(wallWidth);
var frontWall = createWall(wallWidth);
var backWall = createWall(wallWidth); 

frontWall.position.z = wallWidth / 2;
castle.add(frontWall);

leftWall.rotation.y = -0.5*Math.PI;
leftWall.position.x = -wallWidth / 2;
castle.add(leftWall);

backWall.rotation.y = -1*Math.PI;
backWall.position.z = -wallWidth / 2;
castle.add(backWall);

rightWall.rotation.y = 0.5*Math.PI;
rightWall.position.x = wallWidth / 2;
castle.add(rightWall);

// add gate
var gate = createGate();
castle.add(gate);
gate.position.z = frontWall.position.z;
```

The gate consists of a building (cube), roof (cylinder with 0 top radius and 4 x segments) and the gate itself (2D ellipses for both sides of the building).

```javascript
var gateBuilding = new THREE.Mesh(new THREE.CubeGeometry(gateBuildingWidth, gateBuildingHeight, gateBuildingDepth), wallMaterial);
gateBuilding.castShadow = true;
 
var ellipse = new THREE.EllipseCurve(0, 0, 13, 25, 0, Math.PI);
var ellipsePath = new THREE.Path(ellipse.getPoints(50));

var gateGeometry = new THREE.ShapeGeometry(ellipsePath.toShapes()[0]);
var gateMaterial = new THREE.MeshBasicMaterial({ color: 0x4e3100 });

var outerGate = new THREE.Mesh(gateGeometry, gateMaterial);
var innerGate = new THREE.Mesh(gateGeometry, gateMaterial);

outerGate.position.set(0, -gateBuildingHeight / 2, gateBuildingDepth / 2 + 0.2);

gateBuilding.add(outerGate);

innerGate.rotation.y = Math.PI; // rotate because only one side is rendered
innerGate.position.set(0, -gateBuildingHeight / 2, -gateBuildingDepth / 2 - 0.2);

gateBuilding.add(innerGate);

// create roof
var roof = new THREE.Mesh(new THREE.CylinderGeometry(0, 25, 16, 4), roofMaterial);
roof.castShadow = true;
roof.rotation.y = 0.25*Math.PI;
roof.position.y = gateBuildingHeight / 2 + 8;
gateBuilding.add(roof);
```

The main castle building is a cube with two 2D triangles on top of it (front and back sides) and the roof on the left and right sides. 

```javascript
function createUpperPart() {
    var upperPartGeometry = new THREE.Geometry();
    upperPartGeometry.vertices.push(new THREE.Vector3(-buildingWidth / 2, 0, 0));
    upperPartGeometry.vertices.push(new THREE.Vector3(buildingWidth / 2, 0, 0));
    upperPartGeometry.vertices.push(new THREE.Vector3(0, roofHeight, 0));

    upperPartGeometry.faces.push(new THREE.Face3(0, 1, 2));
    upperPartGeometry.computeFaceNormals();

    var upperPart = new THREE.Mesh(upperPartGeometry, wallMaterial);
    upperPart.castShadow = true;

    return upperPart;
}

var building = new THREE.Mesh(new THREE.CubeGeometry(buildingWidth, buildingHeight, buildingDepth), wallMaterial);
building.castShadow = true; 

var frontUpperBuildingPart = createUpperPart();
var backUpperBuildingPart = createUpperPart();

frontUpperBuildingPart.position.set(0, buildingHeight / 2, buildingDepth / 2); 
building.add(frontUpperBuildingPart);

backUpperBuildingPart.rotation.y = Math.PI; 
backUpperBuildingPart.position.set(0, buildingHeight / 2, -buildingDepth / 2); 
building.add(backUpperBuildingPart);

var rightRoof = createRoof();
var leftRoof = createRoof();

rightRoof.position.set(0, buildingHeight, 0);
rightRoof.position.y = buildingHeight / 2;
building.add(rightRoof);

leftRoof.rotation.y = Math.PI;
leftRoof.position.y = buildingHeight / 2;
building.add(leftRoof);
```

The roof consists of two 2D rectangles, so it probably could be implemented using `PlaneGeometry`, but I was not sure how to rotate it to the need position, so I found it easier to specify vertices coordinates directly.

```javascript
    roofGeometry.vertices.push(new THREE.Vector3(buildingWidth / 2, 0, -buildingDepth / 2));
    roofGeometry.vertices.push(new THREE.Vector3(0, roofHeight, -buildingDepth / 2));
    roofGeometry.vertices.push(new THREE.Vector3(buildingWidth / 2, 0, buildingDepth / 2));
    roofGeometry.vertices.push(new THREE.Vector3(0, roofHeight, buildingDepth / 2));

    roofGeometry.faces.push(new THREE.Face3(0, 1, 2));
    roofGeometry.faces.push(new THREE.Face3(3, 2, 1));
    roofGeometry.computeFaceNormals();
```

It also has a door (2D plane) and windows on front, left and right sides.

The windows are created using ellipse shapes similar to the gate

```javascript
function createWindow(xRadius, yRadius, material) {
    var ellipse = new THREE.EllipseCurve(0, 0, xRadius, yRadius, 0, Math.PI);
    var ellipsePath = new THREE.Path(ellipse.getPoints(50));
    var windowGeometry = new THREE.ShapeGeometry(ellipsePath.toShapes()[0]);
    return new THREE.Mesh(windowGeometry, material);
}
```

and added to the walls in loops like this (front side):

```javascript
for (x = -buildingWidth / 2 + windowMargin; x < buildingWidth / 2 - windowMargin; x += windowMargin) {
    wind = createWindow(windowXRadius, windowYRadius, windowMaterial);

    wind.position.set(x, windowY, buildingDepth / 2 + 0.2);

    building.add(wind);
}
```

The front side of the building also has one bigger window in the upper part:

```javascript
wind = createWindow(8, 24, windowMaterial);
wind.position.set(0, 26, buildingDepth / 2 + 0.2);
building.add(wind);
```
