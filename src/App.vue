<template>
	<div class="app-container">
		<div ref="containerRef" class="canvas-container"></div>
		<div class="info">
			<h3>Three.js 多模型加载 Demo</h3>
			<p>使用右侧控制面板添加和调整模型</p>
			<p>鼠标左键拖动旋转 | 右键拖动平移 | 滚轮缩放</p>
			<p>当前模型数量: {{ models.length }}</p>
		</div>
		<div v-if="loadingCount > 0" class="loading">
			加载模型中... ({{ loadingCount }})
		</div>
	</div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from "vue"
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js"
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js"
import { GUI } from "lil-gui"

const containerRef = ref<HTMLDivElement>()
const loadingCount = ref(0)

// 模型信息接口
interface ModelInfo {
	id: number
	name: string
	group: THREE.Group
	params: {
		position: { x: number; y: number; z: number }
		rotation: { x: number; y: number; z: number }
		scale: number
		autoRotate: boolean
		rotateSpeed: number
		visible: boolean
		material: {
			color: string
			roughness: number
			metalness: number
			emissive: string
			emissiveIntensity: number
		}
	}
	folder?: any
}

const models = ref<ModelInfo[]>([])
let modelIdCounter = 0

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let controls: OrbitControls
let gui: GUI
let modelsFolder: any
let directionalLight: THREE.DirectionalLight
let ambientLight: THREE.AmbientLight
let pointLight: THREE.PointLight

// 预设模型列表
const modelPresets = [
	{
		name: "机器人",
		url: "https://threejs.org/examples/models/gltf/RobotExpressive/RobotExpressive.glb",
	},
	{
		name: "士兵",
		url: "https://threejs.org/examples/models/gltf/Soldier.glb",
	},
	{
		name: "火烈鸟",
		url: "https://threejs.org/examples/models/gltf/Flamingo.glb",
	},
	{
		name: "鹦鹉",
		url: "https://threejs.org/examples/models/gltf/Parrot.glb",
	},
	{
		name: "鹳鸟",
		url: "https://threejs.org/examples/models/gltf/Stork.glb",
	},
]

// 全局光源参数
const lightParams = {
	directionalLight: {
		color: "#ffffff",
		intensity: 1,
		positionX: 5,
		positionY: 5,
		positionZ: 5,
	},
	ambientLight: {
		color: "#ffffff",
		intensity: 0.5,
	},
	pointLight: {
		color: "#ffffff",
		intensity: 1,
		positionX: -3,
		positionY: 3,
		positionZ: 3,
	},
}

// 相机参数
const cameraParams = {
	positionX: 0,
	positionY: 5,
	positionZ: 10,
	fov: 50,
}

// 场景参数
const sceneParams = {
	backgroundColor: "#1a1a1a",
}

const init = () => {
	if (!containerRef.value) return

	// 创建场景
	scene = new THREE.Scene()
	scene.background = new THREE.Color(sceneParams.backgroundColor)

	// 创建相机
	camera = new THREE.PerspectiveCamera(
		cameraParams.fov,
		containerRef.value.clientWidth / containerRef.value.clientHeight,
		0.1,
		1000
	)
	camera.position.set(
		cameraParams.positionX,
		cameraParams.positionY,
		cameraParams.positionZ
	)

	// 创建渲染器
	renderer = new THREE.WebGLRenderer({ antialias: true })
	renderer.setSize(
		containerRef.value.clientWidth,
		containerRef.value.clientHeight
	)
	renderer.setPixelRatio(window.devicePixelRatio)
	renderer.shadowMap.enabled = true
	renderer.shadowMap.type = THREE.PCFSoftShadowMap
	containerRef.value.appendChild(renderer.domElement)

	// 添加轨道控制器
	controls = new OrbitControls(camera, renderer.domElement)
	controls.enableDamping = true
	controls.dampingFactor = 0.05

	// 添加光源
	// 环境光
	ambientLight = new THREE.AmbientLight(
		lightParams.ambientLight.color,
		lightParams.ambientLight.intensity
	)
	scene.add(ambientLight)

	// 平行光
	directionalLight = new THREE.DirectionalLight(
		lightParams.directionalLight.color,
		lightParams.directionalLight.intensity
	)
	directionalLight.position.set(
		lightParams.directionalLight.positionX,
		lightParams.directionalLight.positionY,
		lightParams.directionalLight.positionZ
	)
	directionalLight.castShadow = true
	directionalLight.shadow.mapSize.width = 2048
	directionalLight.shadow.mapSize.height = 2048
	scene.add(directionalLight)

	// 点光源
	pointLight = new THREE.PointLight(
		lightParams.pointLight.color,
		lightParams.pointLight.intensity,
		50
	)
	pointLight.position.set(
		lightParams.pointLight.positionX,
		lightParams.pointLight.positionY,
		lightParams.pointLight.positionZ
	)
	scene.add(pointLight)

	// 添加地面
	const groundGeometry = new THREE.PlaneGeometry(30, 30)
	const groundMaterial = new THREE.MeshStandardMaterial({
		color: "#2a2a2a",
		roughness: 0.8,
		metalness: 0.2,
	})
	const ground = new THREE.Mesh(groundGeometry, groundMaterial)
	ground.rotation.x = -Math.PI / 2
	ground.position.y = -1
	ground.receiveShadow = true
	scene.add(ground)

	// 添加网格辅助线
	const gridHelper = new THREE.GridHelper(30, 30, "#444444", "#222222")
	gridHelper.position.y = -0.99
	scene.add(gridHelper)

	// 添加坐标轴辅助线
	const axesHelper = new THREE.AxesHelper(5)
	scene.add(axesHelper)

	// 创建控制面板
	createGUI()

	// 窗口大小调整
	window.addEventListener("resize", onWindowResize)

	// 开始动画循环
	animate()
}

const loadModel = (modelUrl: string, modelName: string, offsetX = 0) => {
	loadingCount.value++
	const loader = new GLTFLoader()

	loader.load(
		modelUrl,
		gltf => {
			const modelGroup = gltf.scene

			// 设置模型属性
			modelGroup.traverse(child => {
				if ((child as THREE.Mesh).isMesh) {
					child.castShadow = true
					child.receiveShadow = true
				}
			})

			// 调整模型大小和位置
			const box = new THREE.Box3().setFromObject(modelGroup)
			const center = box.getCenter(new THREE.Vector3())
			const size = box.getSize(new THREE.Vector3())

			// 缩放模型使其高度约为2单位
			const maxDim = Math.max(size.x, size.y, size.z)
			const scale = 2 / maxDim
			modelGroup.scale.multiplyScalar(scale)

			// 将模型中心移到原点
			modelGroup.position.x = -center.x * scale + offsetX
			modelGroup.position.y = -center.y * scale
			modelGroup.position.z = -center.z * scale

			scene.add(modelGroup)

			// 创建模型信息
			const modelInfo: ModelInfo = {
				id: modelIdCounter++,
				name: `${modelName} #${modelIdCounter}`,
				group: modelGroup,
				params: {
					position: {
						x: modelGroup.position.x,
						y: modelGroup.position.y,
						z: modelGroup.position.z,
					},
					rotation: { x: 0, y: 0, z: 0 },
					scale: 1,
					autoRotate: false,
					rotateSpeed: 1,
					visible: true,
					material: {
						color: "#ffffff",
						roughness: 0.5,
						metalness: 0.5,
						emissive: "#000000",
						emissiveIntensity: 0,
					},
				},
			}

			models.value.push(modelInfo)
			createModelControls(modelInfo)

			loadingCount.value--
			console.log(`模型 ${modelInfo.name} 加载成功！`)
		},
		progress => {
			const percent = (progress.loaded / progress.total) * 100
			console.log(`加载进度: ${percent.toFixed(2)}%`)
		},
		error => {
			console.error("模型加载失败:", error)
			loadingCount.value--
		}
	)
}

const createGUI = () => {
	gui = new GUI()
	gui.title("控制面板")

	// 模型添加器
	const addModelFolder = gui.addFolder("➕ 添加模型")
	const addModelParams = {
		selectedModel: modelPresets[0]?.name || "",
		addModel: () => {
			const preset = modelPresets.find(
				p => p.name === addModelParams.selectedModel
			)
			if (preset) {
				const offsetX = models.value.length * 3
				loadModel(preset.url, preset.name, offsetX)
			}
		},
		customUrl: "",
		customName: "自定义模型",
		addCustomModel: () => {
			if (addModelParams.customUrl.trim()) {
				const offsetX = models.value.length * 3
				loadModel(addModelParams.customUrl, addModelParams.customName, offsetX)
			} else {
				alert("请输入模型URL")
			}
		},
	}

	addModelFolder
		.add(
			addModelParams,
			"selectedModel",
			modelPresets.map(p => p.name)
		)
		.name("选择预设模型")
	addModelFolder.add(addModelParams, "addModel").name("🎯 添加预设模型")
	addModelFolder.add(addModelParams, "customUrl").name("自定义URL")
	addModelFolder.add(addModelParams, "customName").name("自定义名称")
	addModelFolder.add(addModelParams, "addCustomModel").name("🎯 添加自定义模型")
	addModelFolder.open()

	// 模型列表文件夹
	modelsFolder = gui.addFolder("📦 模型列表")
	modelsFolder.open()

	// 相机控制
	const cameraFolder = gui.addFolder("📷 相机控制")
	cameraFolder
		.add(cameraParams, "positionX", -20, 20, 0.1)
		.name("X 位置")
		.onChange(updateCamera)
	cameraFolder
		.add(cameraParams, "positionY", -20, 20, 0.1)
		.name("Y 位置")
		.onChange(updateCamera)
	cameraFolder
		.add(cameraParams, "positionZ", -20, 20, 0.1)
		.name("Z 位置")
		.onChange(updateCamera)
	cameraFolder
		.add(cameraParams, "fov", 20, 100, 1)
		.name("视野")
		.onChange(updateCamera)

	// 光源控制
	const lightsFolder = gui.addFolder("💡 光源控制")

	const dirLightFolder = lightsFolder.addFolder("平行光")
	dirLightFolder
		.addColor(lightParams.directionalLight, "color")
		.name("颜色")
		.onChange(updateLights)
	dirLightFolder
		.add(lightParams.directionalLight, "intensity", 0, 3, 0.1)
		.name("强度")
		.onChange(updateLights)
	dirLightFolder
		.add(lightParams.directionalLight, "positionX", -10, 10, 0.5)
		.name("X 位置")
		.onChange(updateLights)
	dirLightFolder
		.add(lightParams.directionalLight, "positionY", -10, 10, 0.5)
		.name("Y 位置")
		.onChange(updateLights)
	dirLightFolder
		.add(lightParams.directionalLight, "positionZ", -10, 10, 0.5)
		.name("Z 位置")
		.onChange(updateLights)

	const ambLightFolder = lightsFolder.addFolder("环境光")
	ambLightFolder
		.addColor(lightParams.ambientLight, "color")
		.name("颜色")
		.onChange(updateLights)
	ambLightFolder
		.add(lightParams.ambientLight, "intensity", 0, 2, 0.1)
		.name("强度")
		.onChange(updateLights)

	const pointLightFolder = lightsFolder.addFolder("点光源")
	pointLightFolder
		.addColor(lightParams.pointLight, "color")
		.name("颜色")
		.onChange(updateLights)
	pointLightFolder
		.add(lightParams.pointLight, "intensity", 0, 3, 0.1)
		.name("强度")
		.onChange(updateLights)
	pointLightFolder
		.add(lightParams.pointLight, "positionX", -10, 10, 0.5)
		.name("X 位置")
		.onChange(updateLights)
	pointLightFolder
		.add(lightParams.pointLight, "positionY", -10, 10, 0.5)
		.name("Y 位置")
		.onChange(updateLights)
	pointLightFolder
		.add(lightParams.pointLight, "positionZ", -10, 10, 0.5)
		.name("Z 位置")
		.onChange(updateLights)

	// 场景控制
	const sceneFolder = gui.addFolder("🌍 场景设置")
	sceneFolder
		.addColor(sceneParams, "backgroundColor")
		.name("背景颜色")
		.onChange(() => {
			scene.background = new THREE.Color(sceneParams.backgroundColor)
		})
}

const createModelControls = (modelInfo: ModelInfo) => {
	if (!modelsFolder) return

	const folder = modelsFolder.addFolder(modelInfo.name)
	modelInfo.folder = folder

	// 位置控制
	const positionFolder = folder.addFolder("位置")
	positionFolder
		.add(modelInfo.params.position, "x", -10, 10, 0.1)
		.onChange(() => updateModelTransform(modelInfo))
	positionFolder
		.add(modelInfo.params.position, "y", -10, 10, 0.1)
		.onChange(() => updateModelTransform(modelInfo))
	positionFolder
		.add(modelInfo.params.position, "z", -10, 10, 0.1)
		.onChange(() => updateModelTransform(modelInfo))

	// 旋转控制
	const rotationFolder = folder.addFolder("旋转")
	rotationFolder
		.add(modelInfo.params.rotation, "x", 0, Math.PI * 2, 0.01)
		.onChange(() => updateModelTransform(modelInfo))
	rotationFolder
		.add(modelInfo.params.rotation, "y", 0, Math.PI * 2, 0.01)
		.onChange(() => updateModelTransform(modelInfo))
	rotationFolder
		.add(modelInfo.params.rotation, "z", 0, Math.PI * 2, 0.01)
		.onChange(() => updateModelTransform(modelInfo))

	// 缩放和其他控制
	folder
		.add(modelInfo.params, "scale", 0.1, 3, 0.1)
		.name("缩放")
		.onChange(() => updateModelTransform(modelInfo))
	folder.add(modelInfo.params, "autoRotate").name("自动旋转")
	folder.add(modelInfo.params, "rotateSpeed", 0.1, 5, 0.1).name("旋转速度")
	folder
		.add(modelInfo.params, "visible")
		.name("显示")
		.onChange(() => {
			modelInfo.group.visible = modelInfo.params.visible
		})

	// 材质控制
	const materialFolder = folder.addFolder("材质")
	materialFolder
		.addColor(modelInfo.params.material, "color")
		.name("颜色")
		.onChange(() => updateMaterial(modelInfo))
	materialFolder
		.add(modelInfo.params.material, "roughness", 0, 1, 0.01)
		.name("粗糙度")
		.onChange(() => updateMaterial(modelInfo))
	materialFolder
		.add(modelInfo.params.material, "metalness", 0, 1, 0.01)
		.name("金属度")
		.onChange(() => updateMaterial(modelInfo))
	materialFolder
		.addColor(modelInfo.params.material, "emissive")
		.name("发光颜色")
		.onChange(() => updateMaterial(modelInfo))
	materialFolder
		.add(modelInfo.params.material, "emissiveIntensity", 0, 2, 0.1)
		.name("发光强度")
		.onChange(() => updateMaterial(modelInfo))

	// 删除按钮
	folder
		.add(
			{
				remove: () => removeModel(modelInfo),
			},
			"remove"
		)
		.name("🗑️ 删除模型")

	folder.open()
}

const removeModel = (modelInfo: ModelInfo) => {
	// 从场景中移除
	scene.remove(modelInfo.group)

	// 从数组中移除
	const index = models.value.findIndex(m => m.id === modelInfo.id)
	if (index !== -1) {
		models.value.splice(index, 1)
	}

	// 销毁GUI文件夹
	if (modelInfo.folder) {
		modelInfo.folder.destroy()
	}

	console.log(`模型 ${modelInfo.name} 已删除`)
}

const updateModelTransform = (modelInfo: ModelInfo) => {
	modelInfo.group.position.set(
		modelInfo.params.position.x,
		modelInfo.params.position.y,
		modelInfo.params.position.z
	)

	modelInfo.group.rotation.set(
		modelInfo.params.rotation.x,
		modelInfo.params.rotation.y,
		modelInfo.params.rotation.z
	)

	const box = new THREE.Box3().setFromObject(modelInfo.group)
	const size = box.getSize(new THREE.Vector3())
	const maxDim = Math.max(size.x, size.y, size.z)
	const scale = (2 / maxDim) * modelInfo.params.scale

	modelInfo.group.scale.set(scale, scale, scale)
}

const updateMaterial = (modelInfo: ModelInfo) => {
	modelInfo.group.traverse(child => {
		if ((child as THREE.Mesh).isMesh) {
			const mesh = child as THREE.Mesh
			if (mesh.material) {
				const material = mesh.material as THREE.MeshStandardMaterial
				material.color.set(modelInfo.params.material.color)
				material.roughness = modelInfo.params.material.roughness
				material.metalness = modelInfo.params.material.metalness
				material.emissive.set(modelInfo.params.material.emissive)
				material.emissiveIntensity = modelInfo.params.material.emissiveIntensity
			}
		}
	})
}

const updateCamera = () => {
	camera.position.set(
		cameraParams.positionX,
		cameraParams.positionY,
		cameraParams.positionZ
	)
	camera.fov = cameraParams.fov
	camera.updateProjectionMatrix()
}

const updateLights = () => {
	directionalLight.color.set(lightParams.directionalLight.color)
	directionalLight.intensity = lightParams.directionalLight.intensity
	directionalLight.position.set(
		lightParams.directionalLight.positionX,
		lightParams.directionalLight.positionY,
		lightParams.directionalLight.positionZ
	)

	ambientLight.color.set(lightParams.ambientLight.color)
	ambientLight.intensity = lightParams.ambientLight.intensity

	pointLight.color.set(lightParams.pointLight.color)
	pointLight.intensity = lightParams.pointLight.intensity
	pointLight.position.set(
		lightParams.pointLight.positionX,
		lightParams.pointLight.positionY,
		lightParams.pointLight.positionZ
	)
}

const animate = () => {
	requestAnimationFrame(animate)

	// 自动旋转所有启用了自动旋转的模型
	models.value.forEach(modelInfo => {
		if (modelInfo.params.autoRotate) {
			modelInfo.group.rotation.y += 0.01 * modelInfo.params.rotateSpeed
			modelInfo.params.rotation.y = modelInfo.group.rotation.y % (Math.PI * 2)
		}
	})

	controls.update()
	renderer.render(scene, camera)
}

const onWindowResize = () => {
	if (!containerRef.value) return

	camera.aspect =
		containerRef.value.clientWidth / containerRef.value.clientHeight
	camera.updateProjectionMatrix()
	renderer.setSize(
		containerRef.value.clientWidth,
		containerRef.value.clientHeight
	)
}

onMounted(() => {
	init()
	// 默认加载第一个模型
	if (modelPresets[0]) {
		loadModel(modelPresets[0].url, modelPresets[0].name, 0)
	}
})

onUnmounted(() => {
	window.removeEventListener("resize", onWindowResize)

	if (gui) {
		gui.destroy()
	}

	if (renderer) {
		renderer.dispose()
	}

	if (containerRef.value && renderer && renderer.domElement) {
		containerRef.value.removeChild(renderer.domElement)
	}
})
</script>

<style scoped>
.app-container {
	position: relative;
	width: 100vw;
	height: 100vh;
	overflow: hidden;
	margin: 0;
	padding: 0;
}

.canvas-container {
	width: 100%;
	height: 100%;
}

.info {
	position: absolute;
	top: 20px;
	left: 20px;
	background: rgba(0, 0, 0, 0.7);
	color: white;
	padding: 15px 20px;
	border-radius: 8px;
	font-family: "Arial", sans-serif;
	pointer-events: none;
	backdrop-filter: blur(10px);
}

.info h3 {
	margin: 0 0 10px 0;
	font-size: 18px;
	font-weight: bold;
}

.info p {
	margin: 5px 0;
	font-size: 14px;
	opacity: 0.9;
}

.loading {
	position: absolute;
	top: 50%;
	left: 50%;
	transform: translate(-50%, -50%);
	background: rgba(0, 0, 0, 0.8);
	color: white;
	padding: 20px 40px;
	border-radius: 10px;
	font-size: 18px;
	font-family: "Arial", sans-serif;
	backdrop-filter: blur(10px);
}
</style>
