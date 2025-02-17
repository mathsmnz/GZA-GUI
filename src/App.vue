<template>
  <div id="container" ref="container"></div>
</template>

<script>
import { ref, onMounted } from "vue";
import * as WEBIFC from "web-ifc";
import * as OBC from "@thatopen/components";
import * as OBCF from "@thatopen/components-front";

export default {
  name: "IfcLoader",
  setup() {
    const container = ref(null);
    const fragments = ref(null);

    const loadIfc = async () => {
      const components = new OBC.Components();
      const worlds = components.get(OBC.Worlds);

      // Create a new world
      const world = worlds.create(
        OBC.SimpleScene,
        OBC.SimpleCamera,
        OBC.SimpleRenderer
      );

      world.scene = new OBC.SimpleScene(components);
      world.renderer = new OBC.SimpleRenderer(components, container.value);
      world.camera = new OBC.SimpleCamera(components);

      components.init();

      world.camera.controls.setLookAt(12, 6, 8, 0, 0, -10);

      world.scene.setup();

      const grids = components.get(OBC.Grids);
      grids.create(world);

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

      const file = await fetch("https://thatopen.github.io/engine_components/resources/small.ifc");
      const data = await file.arrayBuffer();
      const buffer = new Uint8Array(data);
      const model = await fragmentIfcLoader.load(buffer);
      model.name = "example";
      world.scene.three.add(model);

      const highlighter = components.get(OBCF.Highlighter);
      highlighter.setup({ world });
      highlighter.zoomToSelection = true;

      fragments.value = components.get(OBC.FragmentsManager);


    };

    onMounted(() => {
      loadIfc();
    });

    return { container };
  },
};
</script>

<style scoped>
#container {
  width: 100vh;
  height: 100vh;
  margin: 0;
  padding: 0;
}
</style>