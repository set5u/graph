<template lang="pug">
div
  .graph(ref="graphView")
  button.btn.btn-light.position-fixed.top-0.end-0.m-2.toggle-button(data-bs-toggle="collapse" data-bs-target="#settings-collapse") Settings
  #settings-collapse.collapse.position-fixed.bottom-0.bg-white.w-100
    .m-4
      label.h5(for="#input-formula") 数式(0 =)
      input#input-formula.form-control.font-monospace(v-model="data.formula")
      .form-text GLSLの記法に従う必要があります。
      .h5 定数
      .mb-2(v-for="uniform in data.uniforms")
        .form-text 定数名
        .input-group
          input.form-control.font-monospace(v-model="uniform.name")
          button.btn.btn-danger(@click="data.uniforms = data.uniforms.filter((x)=>x!==uniform)") Delete
        .form-text 最小値/値/最大値 (={{uniform.value}})
        .input-group
          input.form-control.font-monospace(v-model.number="uniform.min")
          input.form-control.font-monospace(v-model.number="uniform.value" type="range" :min="uniform.min" :max="uniform.max" :step="(uniform.max-uniform.min)/1000")
          input.form-control.font-monospace(v-model.number="uniform.max")
      .input-group
        input.form-control(v-model="newConstantValueName" placeholder="定数名")
        button.btn.btn-primary(@click="newConstantValueName && newConstantValue()") 定数を追加
            
</template>

<script setup lang="ts">
import {
  Mesh,
  OrthographicCamera,
  PlaneBufferGeometry,
  Scene,
  ShaderMaterial,
  Vector2,
  WebGLRenderer,
} from "three";

const props = defineProps<{ formula: string }>();
const data: {
  formula: string;
  uniforms: { name: string; value: number; min: number; max: number }[];
} = reactive({
  formula: props.formula || "y*y-x*x*x+a*x+b",
  uniforms: [
    {
      name: "a",
      value: 4.1,
      min: -5,
      max: 5,
    },
    {
      name: "b",
      value: -2.3,
      min: -5,
      max: 5,
    },
  ],
});

const newConstantValueName = ref("");
const newConstantValue = () => {
  data.uniforms.push({
    name: newConstantValueName.value,
    value: 0,
    min: -5,
    max: 5,
  });
  newConstantValueName.value = "";
};

const graphView = ref<HTMLDivElement>();

const o: {
  renderer?: WebGLRenderer;
  camera?: OrthographicCamera;
  scene?: Scene;
  geometory?: PlaneBufferGeometry;
  material1?: ShaderMaterial;
  material2?: ShaderMaterial;
  material3?: ShaderMaterial;
  mesh1?: Mesh;
  mesh2?: Mesh;
  mesh3?: Mesh;
} = new Proxy(
  {},
  {
    set(obj, prop, val) {
      if (prop in obj && typeof obj[prop].dispose === "function") {
        obj[prop].dispose();
      }
      obj[prop] = val;
      return true;
    },
    deleteProperty(obj, prop) {
      if (prop in obj) {
        delete obj[prop];
      }
      return true;
    },
  }
);
const finalize = () => {
  for (const key in o) {
    if (typeof o[key].dispose === "function") {
      o[key].dispose();
    }
  }
};

const setViewport = () => {
  o.renderer.setSize(graphView.value.offsetWidth, graphView.value.offsetHeight);
  o.material1.uniforms.resolution.value.x = graphView.value.offsetWidth;
  o.material1.uniforms.resolution.value.y = graphView.value.offsetHeight;
};
const resize = () => {
  setViewport();
};
window.addEventListener("resize", resize);
onMounted(() => {
  o.renderer = new WebGLRenderer();
  o.renderer.getContext().disable(o.renderer.getContext().DEPTH_TEST);
  graphView.value.appendChild(o.renderer.domElement);

  let moving = false;
  let centerX = 0;
  let centerY = 0;
  let scaleX = 1;
  let scaleY = 1;
  let scale = 0.001;
  let wheelScale = -3;
  graphView.value.addEventListener("mousedown", (e) => {
    moving = true;
  });
  graphView.value.addEventListener("mousemove", (e) => {
    if (moving) {
      centerX = centerX - scaleX * scale * e.movementX * 2;
      centerY = centerY + scaleY * scale * e.movementY * 2;
    }
  });
  graphView.value.addEventListener("mouseup", (e) => {
    moving = false;
  });
  graphView.value.addEventListener("wheel", (e) => {
    e.preventDefault();
    wheelScale += e.deltaY / 1000;
    scale = Math.pow(10, wheelScale);
  });
  o.camera = new OrthographicCamera(-1, 1, 1, -1, 0.1, 2);
  o.camera.position.z = 1;
  o.camera.lookAt(0, 0, 0);
  o.scene = new Scene();
  o.geometory = new PlaneBufferGeometry(2, 2);
  const createLineMaterial1 = () =>
    new ShaderMaterial({
      vertexShader: `
      varying vec2 v_UV;
      void main() {
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
        v_UV = uv;
      }`,
      fragmentShader: `
      varying vec2 v_UV;
      uniform float left;
      uniform float right;
      uniform float top;
      uniform float bottom;
      uniform vec2 resolution;
      uniform float lineWidth;
      uniform float res;
      const float PI = 3.14159265359;
      float func(float x, float y) {
        float vmin = min(right-left,top-bottom);
        return sin(PI*x*res) * sin(PI*y*res);
      }
      void main() {
        float x = left + v_UV.x * (right - left);
        float y = bottom + v_UV.y * (top - bottom);
        vec2 pixelSize = 1./resolution;
        float s = abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x+pixelSize.y*lineWidth*(top-bottom),y))/2.+1.)) + 
                  abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x,y+pixelSize.x*lineWidth*(right-left)))/2.+1.));
        if(s>.5){
          gl_FragColor = vec4(1.,1.,1.,1.);
        }else{
          discard;
        }
      }`,
      uniforms: {
        left: { value: -1 },
        right: { value: 1 },
        top: { value: 1 },
        bottom: { value: -1 },
        resolution: { value: new Vector2(512, 512) },
        lineWidth: { value: 3 },
        res: { value: 10 },
      },
    });
  o.material1 = createLineMaterial1();
  o.mesh1 = new Mesh(o.geometory, o.material1);
  o.mesh1.renderOrder = 1;
  o.scene.add(o.mesh1);
  const createLineMaterial2 = () =>
    new ShaderMaterial({
      vertexShader: `
      varying vec2 v_UV;
      void main() {
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
        v_UV = uv;
      }`,
      fragmentShader: `
      varying vec2 v_UV;
      uniform float left;
      uniform float right;
      uniform float top;
      uniform float bottom;
      uniform vec2 resolution;
      uniform float lineWidth;
      uniform float res;
      const float PI = 3.14159265359;
      float func(float x, float y) {
        float vmin = min(right-left,top-bottom);
        return sin(PI*x*res) * sin(PI*y*res);
      }
      void main() {
        float x = left + v_UV.x * (right - left);
        float y = bottom + v_UV.y * (top - bottom);
        vec2 pixelSize = 1./resolution;
        float s = abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x+pixelSize.y*lineWidth*(top-bottom),y))/2.+1.)) + 
                  abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x,y+pixelSize.x*lineWidth*(right-left)))/2.+1.));
        if(s>.5){
          gl_FragColor = vec4(.5,.5,.5,1.);
        }else{
          discard;
        }
      }`,
      uniforms: {
        left: { value: -1 },
        right: { value: 1 },
        top: { value: 1 },
        bottom: { value: -1 },
        resolution: { value: new Vector2(512, 512) },
        lineWidth: { value: 0.5 },
        res: { value: 10 },
      },
    });
  o.material2 = createLineMaterial2();
  o.mesh2 = new Mesh(o.geometory, o.material2);
  o.mesh2.renderOrder = 2;
  o.scene.add(o.mesh2);

  const createLineMaterial3 = () =>
    new ShaderMaterial({
      vertexShader: `
      varying vec2 v_UV;
      void main() {
        gl_Position = projectionMatrix * modelViewMatrix * vec4( position, 1.0 );
        v_UV = uv;
      }`,
      fragmentShader: `
      varying vec2 v_UV;
      uniform float left;
      uniform float right;
      uniform float top;
      uniform float bottom;
      uniform vec2 resolution;
      uniform float lineWidth;
      ${(() => {
        let ret = "";
        for (const current of data.uniforms) {
          ret += `uniform float ${current.name};`;
        }
        return ret;
      })()}
      const float PI = 3.14159265359;
      float func(float x, float y) {
        float vmin = min(right-left,top-bottom);
        return ${data.formula};
      }
      void main() {
        float x = left + v_UV.x * (right - left);
        float y = bottom + v_UV.y * (top - bottom);
        vec2 pixelSize = 1./resolution;
        float s = abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x+pixelSize.y*lineWidth*(top-bottom),y))/2.+1.)) + 
                  abs(floor(sign(func(x,y))/2.+1.)-floor(sign(func(x,y+pixelSize.x*lineWidth*(right-left)))/2.+1.));
        if(s>.5){
          gl_FragColor = vec4(0.,1.,1.,1.);
        }else{
          discard;
        }
      }`,
      uniforms: {
        left: { value: -1 },
        right: { value: 1 },
        top: { value: 1 },
        bottom: { value: -1 },
        resolution: { value: new Vector2(512, 512) },
        lineWidth: { value: 2 },
        ...(() => {
          const ret: { [key: string]: { value: number } } = {};
          for (const current of data.uniforms) {
            ret[current.name] = { value: current.value };
          }
          return ret;
        })(),
      },
    });
  o.material3 = createLineMaterial3();
  o.mesh3 = new Mesh(o.geometory, o.material3);
  o.mesh3.renderOrder = 3;
  o.scene.add(o.mesh3);

  let timeoutId: number;
  watch(
    data,
    () => {
      for (const uniform of data.uniforms) {
        if (uniform.name in o.material3.uniforms) {
          o.material3.uniforms[uniform.name].value = uniform.value;
        }
      }
      if (timeoutId) clearTimeout(timeoutId);
      timeoutId = window.setTimeout(() => {
        o.mesh3.material = o.material3 = createLineMaterial3();
      }, 1000);
    },
    { deep: true }
  );

  const render = () => {
    requestAnimationFrame(render);
    o.material1.uniforms.left.value =
      centerX - graphView.value.offsetWidth * scaleX * scale;
    o.material1.uniforms.right.value =
      centerX + graphView.value.offsetWidth * scaleX * scale;
    o.material1.uniforms.top.value =
      centerY + graphView.value.offsetHeight * scaleY * scale;
    o.material1.uniforms.bottom.value =
      centerY - graphView.value.offsetHeight * scaleY * scale;
    o.material1.uniforms.res.value = Math.pow(
      10,
      -Math.floor(
        Math.log10(
          Math.max(
            graphView.value.offsetWidth * scaleX * scale * 2,
            graphView.value.offsetHeight * scaleY * scale * 2
          )
        )
      )
    );
    o.material2.uniforms.left.value =
      centerX - graphView.value.offsetWidth * scaleX * scale;
    o.material2.uniforms.right.value =
      centerX + graphView.value.offsetWidth * scaleX * scale;
    o.material2.uniforms.top.value =
      centerY + graphView.value.offsetHeight * scaleY * scale;
    o.material2.uniforms.bottom.value =
      centerY - graphView.value.offsetHeight * scaleY * scale;
    o.material2.uniforms.res.value = Math.pow(
      10,
      -Math.floor(
        Math.log10(
          Math.max(
            graphView.value.offsetWidth * scaleX * scale * 2,
            graphView.value.offsetHeight * scaleY * scale * 2
          )
        )
      ) + 1
    );
    o.material3.uniforms.left.value =
      centerX - graphView.value.offsetWidth * scaleX * scale;
    o.material3.uniforms.right.value =
      centerX + graphView.value.offsetWidth * scaleX * scale;
    o.material3.uniforms.top.value =
      centerY + graphView.value.offsetHeight * scaleY * scale;
    o.material3.uniforms.bottom.value =
      centerY - graphView.value.offsetHeight * scaleY * scale;
    o.renderer.render(o.scene, o.camera);
  };
  setViewport();

  render();
});
onUnmounted(() => {
  finalize();
  window.removeEventListener("resize", resize);
});
</script>

<style scoped lang="scss">
.graph {
  height: 100vh;
  background-color: black;
}
.collapse {
  max-height: 50vh;
  overflow-y: auto;
}
.toggle-button {
  z-index: 10000;
}
</style>
