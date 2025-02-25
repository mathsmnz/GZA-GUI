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
    const plans = ref(null);
    const culler = ref(null);
    let modelItems = [];
    let classifier = null;
    let grid = null;

    // Setup world (scene, camera, renderer, etc.)
    const setupWorld = async () => {
      if (!container.value) {
        console.error("Container element is not available!");
        return;
      }

      const worlds = components.get(OBC.Worlds);

      // Create a new world and assign it to the world variable
      world = worlds.create(OBC.SimpleScene, OBC.OrthoPerspectiveCamera, OBCF.PostproductionRenderer);

      world.scene = new OBC.SimpleScene(components);
      world.renderer = new OBCF.PostproductionRenderer(components, container.value); // Ensure container is passed
      world.camera = new OBC.OrthoPerspectiveCamera(components);

      world.renderer.postproduction.enabled = true;
      world.renderer.postproduction.customEffects.outlineEnabled = true;

      components.init();

      world.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);

      world.scene.setup();
      world.scene.three.background = new THREE.Color(0xFFFFFF);
    };

    const captureScreenshot = () => {
      if (!world || !world.renderer || !container.value) return;

      // Esconde o grid antes da captura
      if (grid) grid.three.visible = false;

      // Renderiza a cena
      world.renderer.three.render(world.scene.three, world.camera.three);

      const screenshot = container.value.querySelector('canvas').toDataURL('image/png');
      const link = document.createElement('a');
      link.href = screenshot;
      link.download = `${fileName}.png`;
      link.click();

      // Restaura a visibilidade do grid após a captura
      if (grid) grid.three.visible = true;
    };

    // Setup grid
    const setupGrid = (world) => {
      const grids = components.get(OBC.Grids);
      grid = grids.create(world);

      grid.three.position.y -= 1.5;
      grid.config.color.setHex(0x000000);

      world.renderer.postproduction.customEffects.excludedMeshes.push(grid.three);
    };

    const setupBoundingBox = (modelToFit) => {
      const fragmentBox = components.get(OBC.BoundingBoxer);

      if (modelToFit) {
        fragmentBox.add(modelToFit);

        // Obter o mesh da caixa delimitadora
        const bboxMesh = fragmentBox.getMesh();

        // Garantir que a bounding box seja computada corretamente
        bboxMesh.geometry.computeBoundingBox();
        const boundingBox = bboxMesh.geometry.boundingBox;

        // Calcular o centro e o tamanho da caixa delimitadora
        const center = new THREE.Vector3();
        boundingBox.getCenter(center);

        const size = new THREE.Vector3();
        boundingBox.getSize(size);

        // Calcular a distância ideal da câmera para enquadrar o modelo
        const fov = world.camera.three.fov * (Math.PI / 180); // Converter FOV para radianos
        const maxDim = Math.max(size.x, size.y, size.z);
        let cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2));

        cameraZ *= 1.1; // Fator de segurança

        // Definir a posição da câmera e direcioná-la ao centro do modelo
        world.camera.three.position.set(center.x, center.y, cameraZ);
        world.camera.three.lookAt(center);

        // Ajustar os controles da câmera
        if (world.camera.controls) {
          world.camera.controls.target = [center.x, center.y, center.z];
          world.camera.controls.position = [center.x, center.y, cameraZ];
        }

        fragmentBox.reset(); // Limpar o BoundingBoxer após o uso
      }
    };

    const planManager = async () => {
      plans.value = components.get(OBCF.Plans);
      plans.value.world = world;
      await plans.value.generate(model);



      const cullers = components.get(OBC.Cullers);
      culler.value = cullers.create(world);
      for (const fragment of model.items) {
        culler.value.add(fragment.mesh);
      }

      culler.value.needsUpdate = true;

      world.camera.controls.addEventListener("sleep", () => {
        culler.value.needsUpdate = true;
      });

      classifier = components.get(OBC.Classifier);
      const edges = components.get(OBCF.ClipEdges);

      classifier.byModel(model.uuid, model);
      classifier.byEntity(model);

      modelItems = classifier.find({ models: [model.uuid] });

      const thickItems = classifier.find({
        entities: ["IFCWALLSTANDARDCASE", "IFCWALL"],
      });

      const thinItems = classifier.find({
        entities: ["IFCDOOR", "IFCWINDOW", "IFCPLATE", "IFCMEMBER"],
      });

      const grayFill = new THREE.MeshBasicMaterial({ color: "gray", side: 2 });
      const blackLine = new THREE.LineBasicMaterial({ color: "black" });
      const blackOutline = new THREE.MeshBasicMaterial({
        color: "black",
        opacity: 0.5,
        side: 2,
        transparent: true,
      });

      edges.styles.create(
        "thick",
        new Set(),
        world,
        blackLine,
        grayFill,
        blackOutline,
      );

      const frag = fragments.value;

      for (const fragID in thickItems) {
        const foundFrag = frag.list.get(fragID);
        if (!foundFrag) continue;
        const { mesh } = foundFrag;
        edges.styles.list.thick.fragments[fragID] = new Set(thickItems[fragID]);
        edges.styles.list.thick.meshes.add(mesh);
      }

      edges.styles.create("thin", new Set(), world);

      for (const fragID in thinItems) {
        const foundFrag = frag.list.get(fragID);
        if (!foundFrag) continue;
        const { mesh } = foundFrag;
        edges.styles.list.thin.fragments[fragID] = new Set(thinItems[fragID]);
        edges.styles.list.thin.meshes.add(mesh);
      }

      await edges.update(true);
    }
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
      await setupWorld(); // Initialize world
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

      await planManager();
      setupBoundingBox(model);
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

        // Limpar memória do modelo anterior
        if (model) {
          world.scene.three.remove(model); // Remover da cena

          // Percorrer os meshes do modelo e liberar recursos
          model.traverse((child) => {
            if (child.isMesh) {
              child.geometry.dispose(); // Libera a geometria
              if (child.material) {
                if (Array.isArray(child.material)) {
                  child.material.forEach((m) => m.dispose());
                } else {
                  child.material.dispose(); // Libera o material
                }
              }
            }
          });
          model = null; // Limpar referência do modelo
        }

        // Limpar componentes anteriores
        if (plans.value) {
          plans.value.dispose();
          plans.value = null;
        }

        if (culler.value) {
          culler.value = null;
        }

        if (fragments.value) {
          fragments.value.dispose(); // Liberar memória dos fragmentos
          fragments.value = null;
        }

        // Limpar o cache do WebGL
        world.renderer.three.clear();
        world.renderer.three.dispose();

        // Carregar o novo modelo IFC
        const fragmentIfcLoader = components.get(OBC.IfcLoader);
        model = await fragmentIfcLoader.load(buffer); // Atualiza o modelo

        model.name = file.name;
        fileName = file.name;
        world.scene.three.add(model); // Adicionar o novo modelo à cena

        // Inicializar fragmentos após a configuração da cena
        fragments.value = components.get(OBC.FragmentsManager);
        await planManager();

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

    const activatePlan = (plan) => {
      world.renderer.postproduction.customEffects.minGloss = 0.1;
      highlighter.backupColor = new THREE.Color("white");
      world.scene.three.background = new THREE.Color("white");
      const plansComponent = world.components.get(OBCF.Plans);
      plansComponent.goTo(plan.id);
      culler.needsUpdate = true;
    };

    const exitPlanView = () => {
      world.renderer.postproduction.customEffects.minGloss = 0.0;
      highlighter.backupColor = null;
      highlighter.clear();
      classifier.resetColor(classifier.find({ models: [model.uuid] }));
      world.scene.three.background = null;
      const plansComponent = world.components.get(OBCF.Plans);
      plansComponent.exitPlanView();
      culler.needsUpdate = true;
    };

    onMounted(() => {
      if (container.value) {
        setupScene(); // Initialize scene after mounting
      } else {
        console.error("Container element not found when mounted.");
      }
    });

    return { container, loadIFC, saveFile, activatePlan, exitPlanView, plans, captureScreenshot }; // Expose loadIFC to the template
  },
};
</script>

<template>
  <div class="flex flex-col items-center justify-center w-full h-screen bg-gray-50">
    <!-- Viewer Container with 16:9 aspect ratio and 2/3 screen width -->
    <div ref="container" class="viewer-container"></div>

    <!-- IFC Viewer Controls -->
    <div class="w-64 p-4 bg-gray-100 overflow-y-auto">
      <button @click="loadIFC"
        class="w-full text-left bg-blue-500 text-white p-2 mt-4 rounded shadow hover:bg-blue-600">
        Carregar Novo IFC
      </button>
      <button @click="saveFile"
        class="w-full text-left bg-green-500 text-white p-2 mt-4 rounded shadow hover:bg-green-600">
        Salvar IFC
      </button>
      <button @click="captureScreenshot"
        class="w-full text-left bg-purple-500 text-white p-2 mt-4 rounded shadow hover:bg-purple-600">
        Capturar Tela
      </button>
    </div>

    <div class="text-center mt-8 px-4">
      <p class="text-lg text-gray-800">
        Carregar um arquivo IFC para visualizar o modelo em 3D. Use o botão acima para importar um arquivo.
      </p>
    </div>

    <h2 class="text-lg font-semibold mb-2">Planos</h2>

    <!-- Fallback while plans are not loaded -->
    <div v-if="!plans || !plans.list.length" class="text-gray-500">
      Carregando planos...
    </div>

    <!-- Render plans when available -->
    <div v-else>
      <button v-for="plan in plans.list" :key="plan.id"
        class="w-full text-left bg-white p-2 mb-2 rounded shadow hover:bg-gray-200" @click="activatePlan(plan)">
        {{ plan.name }}
      </button>
      <button class="w-full text-left bg-red-500 text-white p-2 mt-4 rounded shadow hover:bg-red-600"
        @click="exitPlanView">
        Sair
      </button>
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
