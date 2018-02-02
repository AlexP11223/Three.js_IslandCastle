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

`shadowBias` is set to `0.001` because I encountered a shadow artifact and after some research ([https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416324%28v=vs.85%29.aspx), [http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js](http://stackoverflow.com/questions/11118199/why-is-the-shadow-in-the-wrong-place-three-js)) found that it is called “**Peter Panning**” and can be fixed by adjusting `shadowBias` value.

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
