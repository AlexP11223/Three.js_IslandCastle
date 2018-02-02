*A simple scene implemented using Three.js following J. Dirksen book "Learning Three.js" several years ago during Computer Graphics course in university. Uses old version of Three.js and other libs, I don't know if it is compatible with modern versions.*

Castle on an island and three moving boats. Speed can be changed via controls in the top right corner (it also depends on FPS).

![](https://i.gyazo.com/3e825eaf4449d7750e51b62a5df741e6.gif)

Can be run here: [https://alexp11223.github.io/Three.js_IslandCastle/index.html](https://alexp11223.github.io/Three.js_IslandCastle/index.html)

# Implementation details

## World

Lake/water is implemented using a 2D plane:

```
var planeGeometry = new THREE.PlaneGeometry(1000, 500);
var planeMaterial = new THREE.MeshLambertMaterial({color: 0x1acffa}); 
var plane = new THREE.Mesh(planeGeometry, planeMaterial);
plane.receiveShadow  = true;
plane.rotation.x = -0.5*Math.PI;
scene.add(plane);
```

Background color is set to light blue to represent sky:

```
renderer.setClearColor(0x9fd2f1, 1.0);
```

`HemisphereLight` + `DirectionalLight` are used to create a sun-like natural light.

`shadowBias` is set to `0.001` because I encountered a shadow artifact shown below and after some research ([https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx), [http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js](http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js)) I found out that it is called “**Peter Panning**” and can be fixed by adjusting `shadowBias` value.

![](https://i.imgur.com/AYlet59.png)

```
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

```
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

```
var boatMaterial = new THREE.MeshLambertMaterial({color: boatColor});

var boat = new THREE.Mesh(new THREE.CubeGeometry(boatWidth, boatHeight, boatDepth), boatMaterial);
boat.castShadow = true;
```

2. A triangle for the sail.

Here also one more triangle with the same size and at almost the same position was added because if a boat is rotated to the opposite direction then a single 2D sail would not cast a shadow (even with `side` property).

```
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

```
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

```
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

```
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

The island is created using cylinder with bottom radius less than top, and green color material.

```
var island = new THREE.Mesh(new THREE.CylinderGeometry(103, 143, 10, 80), 
                             new THREE.MeshLambertMaterial({color: 0x00ff00}));
island.receiveShadow = true;
```

## Castle
