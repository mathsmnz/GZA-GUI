<script>
import { ref, onMounted } from "vue";
import * as WEBIFC from "web-ifc";
import * as THREE from "three";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

export default {
  name: "IfcLoader",
  setup() {
    const container = ref(null); // Container for the Three.js renderer
    const fragments = ref(null); // Initialize fragments as a ref
    let model = null;
    let world = null;
    const components = new OBC.Components();
    const highlighter = components.get(OBCF.Highlighter);
    let fileName = "";

    // Setup world (scene, camera, renderer, etc.)
    const setupWorld = async () => {
      if (!container.value) {
        console.error("Container element is not available!");
        return;
      }

      const worlds = components.get(OBC.Worlds);

      // Create a new world and assign it to the world variable
      world = worlds.create(OBC.SimpleScene, OBC.SimpleCamera, OBC.SimpleRenderer);

      world.scene = new OBC.SimpleScene(components);
      world.renderer = new OBC.SimpleRenderer(components, container.value); // Ensure container is passed
      world.camera = new OBC.SimpleCamera(components);

      components.init();

      world.scene.three.background = new THREE.Color(0xff0000);
      world.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);

      world.scene.setup();

      return world; // Return world to use later
    };

    // Setup grid
    const setupGrid = (world) => {
      const grids = components.get(OBC.Grids);
      grids.create(world);

      const gridHelper = world.scene.three.getObjectByName("grid");
      if (gridHelper) {
        gridHelper.material.color.set(0x000000); // Black color
      }
    };

    // Load the IFC model
    const loadIfcModel = async (url) => {
      const fragmentIfcLoader = components.get(OBC.IfcLoader);
      await fragmentIfcLoader.setup();

      const excludedCats = [
        WEBIFC.IFCTENDONANCHOR,
        WEBIFC.IFCREINFORCINGBAR,
        WEBIFC.IFCREINFORCINGELEMENT,
      ];

      for (const cat of excludedCats) {
        fragmentIfcLoader.settings.excludedCategories.add(cat);
      }

      fragmentIfcLoader.settings.webIfc.COORDINATE_TO_ORIGIN = true;

      const file = await fetch(url);
      const data = await file.arrayBuffer();
      const buffer = new Uint8Array(data);

      const loadedModel = await fragmentIfcLoader.load(buffer);
      loadedModel.name = "example";

      return loadedModel; // Return loaded model
    };

    // Setup the entire scene
    const setupScene = async () => {
      const world = await setupWorld(); // Initialize world
      if (!world) return; // Ensure world is valid before proceeding

      setupGrid(world); // Setup grid in the scene

      // Load and add the IFC model to the scene
      const url = "https://raw.githubusercontent.com/mathsmnz/GZA-GUI/refs/heads/main/public/H.ifc";
      model = await loadIfcModel(url); // Store the loaded model
      world.scene.three.add(model); // Add model to the scene
      fileName = "EXAMPLE"

      highlighter.setup({ world });
      highlighter.zoomToSelection = true;

      // Initialize fragments after the scene is set up
      fragments.value = components.get(OBC.FragmentsManager); // Correctly assign fragments
      console.log(fragments.value);
    };

    // Load IFC model from file input
    const loadIFC = async () => {
      const fileInput = document.createElement('input');
      fileInput.type = 'file';
      fileInput.accept = '.ifc';
      fileInput.onchange = async (e) => {
        const file = e.target.files[0];
        const data = await file.arrayBuffer();
        const buffer = new Uint8Array(data);

        if (!world) {
          console.error("World is not initialized.");
          return;
        }

        world.scene.three.remove(model);

        const fragmentIfcLoader = components.get(OBC.IfcLoader);
        model = await fragmentIfcLoader.load(buffer); // Update model with the newly loaded IFC model
        model.name = file.name;
        fileName = file.name;
        world.scene.three.add(model); // Add the new model to the scene

        // Initialize fragments after the scene is set up
        fragments.value = components.get(OBC.FragmentsManager); // Correctly assign fragments
        console.log(fragments.value);

        highlighter.setup({ world });
        highlighter.zoomToSelection = true;
      };
      fileInput.click();
    };

    // Save .frag and properties.json
    const saveFile = () => {
      const downloadLink = document.createElement('a');
      if (!fragments.value.groups.size) {
        return;
      }
      const group = Array.from(fragments.value.groups.values())[0];
      const data = fragments.value.export(group);
      const name = fileName.split('.')[1];

      const properties = group.getLocalProperties();
      if (properties) {
        const newFile = new File([JSON.stringify(properties)], `${fileName}.json`);
        downloadLink.href = URL.createObjectURL(newFile);
        downloadLink.download = newFile.name;
        downloadLink.click();
      }

      const newFile = new File([new Blob([data])], `${fileName}.frag`);
      downloadLink.href = URL.createObjectURL(newFile);
      downloadLink.download = newFile.name;
      downloadLink.click();

    }

    onMounted(() => {
      if (container.value) {
        setupScene(); // Initialize scene after mounting
      } else {
        console.error("Container element not found when mounted.");
      }
    });

    return { container, loadIFC, saveFile }; // Expose loadIFC to the template
  },
};
</script>

<template>
  <div class="flex flex-col items-center justify-center w-full h-screen bg-gray-50">
    <!-- Viewer Container with 16:9 aspect ratio and 2/3 screen width -->
    <div ref="container" class="viewer-container">
      <!-- IFC Viewer Controls -->
      <div class="controls">
        <button @click="loadIFC" class="load-button">
          Carregar Novo IFC
        </button>
        <button @click="saveFile" class="load-button">
          Salvar IFC
        </button>
      </div>
    </div>

    <!-- Optional Info Section -->
    <div class="text-center mt-8 px-4">
      <p class="text-lg text-gray-800">
        Carregar um arquivo IFC para visualizar o modelo em 3D. Use o botão acima para importar um arquivo.
      </p>
    </div>
  </div>
</template>

<style scoped>
/* Container styles */
/* Container styles */
.viewer-container {
  position: relative;
  width: 66.6666vw;
  /* 2/3 of the screen width */
  height: calc(66.6666vw * 9 / 16);
  /* 16:9 aspect ratio (height = width * 9/16) */
  background-color: white;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  border-radius: 8px;
  overflow: hidden;
}
</style>
