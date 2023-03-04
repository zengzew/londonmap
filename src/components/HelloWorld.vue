<template>
  <div id="hello">
    <div id="cont">
      <div id="panel">
        <ul id="list">
          <li v-for="(value, key) in Table_Info" :key="value">
            {{ key }}: {{ value }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
import * as THREE from "three";
import { MapControls } from "three/addons/controls/OrbitControls";
import * as GEOLIB from "geolib";
import * as GEOTIFF from "geotiff";
import Stats from "three/examples/jsm/libs/stats.module.js";
import { mergeBufferGeometries } from "three/addons/utils/BufferGeometryUtils.js";
import { Water } from "three/examples/jsm/objects/Water";

export default {
  name: "HelloWorld",
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      controls: null,
      stats: null,
      raycaster: null,
      //center: [114.1722, 22.2974],
      center: [-0.1043, 51.5098], //london

      iR: null,
      iR_Line: [],
      iR_Water: [],
      geos_building: [],
      MAT_BUILDING: null,
      MAT_ROAD: null,
      MAT_WATER_NORMAL: null,
      MAT_WATER: null,
      collider_building: [],
      FLAG_ROAD_ANI: true,
      Animated_Line_Distances: [],

      Table_Info: {
        name: "",
        housenumber: "",
        street: "",
        levels: "",
        postcode: "",
      },
    };
  },
  mounted() {
    this.Awake();
    let that = this;

    window.addEventListener("resize", onWindowResize, false);
    function onWindowResize() {
      that.camera.aspect = window.innerWidth / window.innerHeight;
      that.camera.updateProjectionMatrix();
      that.renderer.setSize(window.innerWidth, window.innerHeight);
    }
    onWindowResize();
    document.getElementById("cont").addEventListener("mousedown", (evt) => {
      let mouse = {
        x: (evt.clientX / window.innerWidth) * 2 - 1,
        y: -(evt.clientY / window.innerHeight) * 2 + 1,
      };

      let hitted = that.Fire(mouse);
      if (hitted["info"]) {
        let inf = hitted.info;
        console.log(inf);
        //this.Table_Info = hitted.info;
        if (inf.name) {
          this.Table_Info.name = inf.name;
        } else {
          this.Table_Info.name = inf["addr:housename"];
        }
        if (inf.housenumber) {
          this.Table_Info.housenumber = inf.housenumber;
        } else {
          this.Table_Info.housenumber = inf["addr:housenumber"];
        }
        this.Table_Info.street = inf["addr:street"];
        this.Table_Info.levels = inf["building:levels"];
        this.Table_Info.postcode = inf["addr:postcode"];
      }
    });
  },

  methods: {
    Awake() {
      let cont = document.getElementById("cont");

      // Init scene
      this.scene = new THREE.Scene();

      this.scene.background = new THREE.Color(0x222222);

      // Init Camera
      this.camera = new THREE.PerspectiveCamera(
        100,
        window.clientWidth / window.clientHeight,
        1,
        100
      );
      this.camera.position.set(6, 10, 0);

      //init raycaster
      this.raycaster = new THREE.Raycaster();

      // Init group
      this.iR = new THREE.Group();
      this.iR.name = "Interactive Root";

      this.iR_Line = new THREE.Group();
      this.iR_Line.name = "Animated Line on Roads";

      this.iR_Water = new THREE.Group();

      this.$nextTick(() => {
        this.scene.add(this.iR);
        this.scene.add(this.iR_Line);
        this.scene.add(this.iR_Water);
      });

      // Init Light
      let light0 = new THREE.AmbientLight(0xfafafa, 0.25);

      let light1 = new THREE.PointLight(0xfafafa, 0.4);
      light1.position.set(200, 90, 40);

      let light2 = new THREE.PointLight(0xfafafa, 0.4);
      light2.position.set(200, 90, -40);

      this.scene.add(light0);
      this.scene.add(light1);
      this.scene.add(light2);

      let gridHelper = new THREE.GridHelper(
        40,
        160,
        new THREE.Color(0x555555),
        new THREE.Color(0x333333)
      );
      this.scene.add(gridHelper);

      // let geometry = new THREE.BoxGeometry(1,1,1)
      // let material = new THREE.MeshBasicMaterial({color: 0x00ff00})
      // let mesh = new THREE.Mesh(geometry, material)
      // this.scene.add(mesh)

      // Init renderer
      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
      });
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.setSize(window.innerWidth, window.innerHeight);

      cont.appendChild(this.renderer.domElement);

      this.controls = new MapControls(this.camera, this.renderer.domElement);
      this.controls.enableDamping = true;
      this.controls.dampingFactor = 0.25;
      this.controls.screenSpacePanning = false;
      this.controls.maxDistance = 800;

      this.controls.update();
      this.stats = new Stats();
      cont.appendChild(this.stats.domElement);
      this.MAT_BUILDING = new THREE.MeshPhongMaterial();

      this.Update();

      this.GetGeoJson();

      //this.LoadTerrain();
    },

    Update() {
      requestAnimationFrame(this.Update);

      this.renderer.render(this.scene, this.camera);
      this.controls.update();

      this.UpdateAnimatedLine();

      this.updateWater();

      this.stats.update();
    },
    GetGeoJson() {
      fetch("./assets/london.geojson").then((res) => {
        res.json().then((data) => {
          this.LoadBuildings(data);
        });
      });
    },
    async LoadTerrain() {
      const rawTiff = await GEOTIFF.fromUrl("./assets/terrain.tif");
      const tifImage = await rawTiff.getImage();

      const start = [-3.2287499999999998, 55.9340277780000008];
      const end = [-3.1437499999999998, 55.9637499999999974];

      let leftBottom = this.GPSRelativePosition(start, this.center);
      let rightTop = this.GPSRelativePosition(end, this.center);

      let x = Math.abs(leftBottom[0] - rightTop[0]);
      let y = Math.abs(leftBottom[1] - rightTop[1]);
      console.log(x, y);

      const geometry = new THREE.PlaneGeometry(x, y, x - 1, y - 1);

      const data = await tifImage.readRasters({
        width: Math.floor(x),
        height: Math.floor(y),
        resampleMethod: "bilinear",
        interleave: true,
      });
      console.log(data);
      console.log(geometry);

      console.time("parseGeom");
      const arr1 = new Array(geometry.attributes.position.count);
      const arr = arr1.fill(1);
      arr.forEach((a, index) => {
        geometry.attributes.position.setZ(index, data[index] / 30);
      });
      console.timeEnd("parseGeom");

      geometry.rotateX(Math.PI / 2);
      geometry.rotateY(Math.PI / 2);
      geometry.rotateZ(Math.PI);

      let plane = new THREE.Mesh(
        geometry,
        new THREE.MeshPhongMaterial({
          color: 0xff9900,
          side: THREE.DoubleSide,
          wireframe: true,
        })
      );
      plane.position.y = -3;
      this.scene.add(plane);
    },
    LoadBuildings(data) {
      let features = data.features;

      this.MAT_ROAD = new THREE.LineBasicMaterial({
        color: 0x254360,
      });

      for (let i = 0; i < features.length; i++) {
        let fel = features[i];
        if (!fel["properties"]) return;

        if (fel.properties["building"]) {
          this.addBuilding(
            fel.geometry.coordinates,
            fel.properties,
            fel.properties["building:levels"]
          );
        } else if (fel.properties["highway"]) {
          if (
            fel.geometry.type == "LineString" &&
            fel.properties["highway"] != "pedestrian" &&
            fel.properties["highway"] != "footway" &&
            fel.properties["highway"] != "path"
          ) {
            this.addRoad(fel.geometry.coordinates, fel.properties);
          }
        }
      }

      let that = this;
      this.$nextTick(() => {
        let mergeGeometry = mergeBufferGeometries(that.geos_building);
        let mesh = new THREE.Mesh(mergeGeometry, that.MAT_BUILDING);
        this.iR.add(mesh);

        that.loadWaters();
      });
    },
    loadWaters() {
      this.MAT_WATER_NORMAL = new THREE.TextureLoader().load(
        "./assets/waternormals.jpg",
        (texture) => {
          texture.wraps = texture.wrapS = texture.wrapT = THREE.RepeatWrapping;
        }
      );
      this.MAT_WATER = {
        textureWidth: 512,
        textureHeight: 512,
        waterNormals: this.MAT_WATER_NORMAL,
        sunDirection: new THREE.Vector3(),
        distortionScale: 3.7,
        sunColor: 0xffffff,
        waterColor: 0x365b81,
        fog: false,
      };

      fetch("./assets/water.geojson").then((res) => {
        res.json().then((data) => {
          let features = data.features;
          for (let i = 0; i < features.length; i++) {
            let fel = features[i];
            // if (
            //   fel.properties.natural == "water" &&
            //   fel.geometry.type == "MultiPolygon" &&
            //   fel.properties.water == "river"
            //)
            //{
            this.addWater(fel.geometry.coordinates);
            // }
          }
        });
      });
    },

    addWater(data) {
      let shape, geometry;
      let holes = [];

      for (let i = 0; i < data.length; i++) {
        let el = data[i];

        if (i == 0) {
          shape = this.genShape(el, this.center);
        } else {
          holes.push(this.genShape(el, this.center));
        }
      }
      for (let i = 0; i < holes.length; i++) {
        shape.holes.push(holes[i]);
      }

      geometry = this.genGeometry(shape, {
        curveSegments: 1,
        depth: 0.1,
        bevelEnabled: false,
      });
      geometry.rotateX(Math.PI / 2);
      geometry.rotateZ(Math.PI);

      let water = new Water(geometry, this.MAT_WATER);
      this.iR_Water.add(water);
    },

    addRoad(data, info) {
      // Init points array
      let points = [];

      // Loop for all nodes
      for (let i = 0; i < data.length; i++) {
        if (!data[0][1]) return;

        let el = data[i];

        //Just in case
        if (!el[0] || !el[1]) return;

        let elp = [el[0], el[1]];

        //convert position from the center position
        elp = this.GPSRelativePosition([elp[0], elp[1]], this.center);

        // Draw Line
        points.push(new THREE.Vector3(elp[0], 0.5, elp[1]));
      }

      let geometry = new THREE.BufferGeometry().setFromPoints(points);

      // Adjust geometry rotation
      geometry.rotateZ(Math.PI);

      let line = new THREE.Line(geometry, this.MAT_ROAD);
      line.info = info;
      line.computeLineDistances();

      this.iR.add(line);
      line.position.set(line.position.x, 0.5, line.position.z);

      if (this.FLAG_ROAD_ANI) {
        // Length of the line
        let lineLength =
          geometry.attributes.lineDistance.array[
            geometry.attributes.lineDistance.count - 1
          ];

        if (lineLength > 0.1) {
          let aniLine = this.addAnimatedLine(geometry, lineLength);
          this.iR_Line.add(aniLine);
        }
      }
    },
    addAnimatedLine(geometry, length) {
      let animatedLine = new THREE.Line(
        geometry,
        new THREE.LineDashedMaterial({ color: 0x00ffff })
      );
      animatedLine.material.transparent = true;
      animatedLine.position.y = 0.5;
      animatedLine.material.dashSize = 0;
      animatedLine.material.gapSize = 1000;

      this.Animated_Line_Distances.push(length);

      return animatedLine;
    },
    UpdateAnimatedLine() {
      // If no animated line than do nothing
      //console.log("update is working");
      if (this.iR_Line.children.length <= 0) return;

      for (let i = 0; i < this.iR_Line.children.length; i++) {
        let line = this.iR_Line.children[i];
        let dash = parseInt(line.material.dashSize);
        let length = parseInt(this.Animated_Line_Distances[i]);

        if (dash > length) {
          line.material.dashSize = 0;
          line.material.opacity = 1;
        } else {
          line.material.dashSize += 0.02;
          line.material.opacity =
            line.material.opacity > 0 ? line.material.opacity - 0.01 : 0;
        }
      }
    },

    addBuilding(data, info, height = 1) {
      height = height ? height : 1;

      let shape, geometry;
      let holes = [];

      for (let i = 0; i < data.length; i++) {
        let el = data[i];
        if (i == 0) {
          shape = this.genShape(el, this.center);
        } else {
          holes.push(this.genShape(el, this.center));
        }

        for (let i = 0; i < holes.length; i++) {
          shape.holes.push(holes[i]);
        }

        let geometry = this.genGeometry(shape, {
          curveSegments: 1,
          depth: 0.05 * height,
          bevelEnabled: false,
        });

        geometry.rotateX(Math.PI / 2);
        geometry.rotateZ(Math.PI);

        this.geos_building.push(geometry);
        //let mesh = new THREE.Mesh(geometry, this.MAT_BUILDING);
        //this.scene.add(mesh);

        let helper = this.genHelper(geometry);
        if (helper) {
          helper.name = info["name"] ? info["name"] : "Building";
          helper.info = info;
          //this.iR.add(helper)
          this.collider_building.push(helper);
        }
      }
    },
    updateWater() {
      for (let i = 0; i < this.iR_Water.children.length; i++) {
        this.iR_Water.children[i].material.uniforms["time"].value += 1.0 / 120;
      }
    },
    genHelper(geometry) {
      if (!geometry.boundingBox) {
        geometry.computeBoundingBox();
      }

      let box3 = geometry.boundingBox;
      if (!isFinite(box3.max.x)) {
        return false;
      }

      let helper = new THREE.Box3Helper(box3, 0xffff00);
      helper.updateMatrixWorld();

      return helper;
    },
    genShape(points, center) {
      let shape = new THREE.Shape();

      for (let i = 0; i < points.length; i++) {
        let elp = points[i];
        elp = this.GPSRelativePosition(elp, center);

        if (i == 0) {
          shape.moveTo(elp[0], elp[1]);
        } else {
          shape.lineTo(elp[0], elp[1]);
        }
      }

      return shape;
    },
    Fire(mouse) {
      this.raycaster.setFromCamera(mouse, this.camera);
      let intersects = this.raycaster.intersectObjects(
        this.collider_building,
        true
      );
      //console.log(intersects);
      if (intersects.length > 0) {
        let selectedObject = intersects[0].object;
        return selectedObject;
      } else {
        return false;
      }
    },
    genGeometry(shape, settings) {
      let geometry = new THREE.ExtrudeGeometry(shape, settings);
      geometry.computeBoundingBox();

      return geometry;
    },

    GPSRelativePosition(objPosi, centerPosi) {
      // Get GPS distance
      let dis = GEOLIB.getDistance(objPosi, centerPosi);

      // Get bearing angle
      let bearing = GEOLIB.getRhumbLineBearing(objPosi, centerPosi);

      // Calculate X by centerPosi.x + distance * cos(rad)
      let x = centerPosi[0] + dis * Math.cos((bearing * Math.PI) / 180);

      // Calculate Y by centerPosi.y + distance * sin(rad)
      let y = centerPosi[1] + dis * Math.sin((bearing * Math.PI) / 180);

      // Reverse X (it work)
      return [-x / 100, y / 100];
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#cont {
  position: absolute;
  height: 100%;
  width: 100%;
}
#panel {
  position: fixed;
  right: 0;
  width: 250px;
  background-color: rgb(34, 55, 53, 0.7);
  color: aliceblue;
}

#list {
  padding: 10px;
  width: 100%;
  list-style-type: none;
}
</style>
