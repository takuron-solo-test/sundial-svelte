<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';

	let container: HTMLDivElement;
	let scene: any;
	let camera: any;
	let renderer: any;
	let controls: any;
	let animationId: number;
	let sundialGroup: any;
	let latitude: number = 39.9; // 北京纬度
	let longitude: number = 116.4; // 北京经度

	// 颜色配置
	const colors = {
		base: 0x8b4513, // 棕色
		disk: 0xfff8dc, // 米白色
		gnomon: 0x000000, // 黑色
		hourMarker: 0x000000, // 黑色
		highlight: 0xff0000, // 红色（当前小时）
		sunlight: 0xffffcc, // 太阳光颜色
		sky: 0x87ceeb // 天空蓝色
	};

	let THREE: any;
	let OrbitControls: any;

	onMount(async () => {
		if (!browser) return;

		// 动态导入Three.js，只在客户端执行
		THREE = await import('three');
		const OrbitControlsModule = await import('three/addons/controls/OrbitControls.js');
		OrbitControls = OrbitControlsModule.OrbitControls;

		init();
		animate();
		window.addEventListener('resize', onWindowResize);
	});

	onDestroy(() => {
		if (!browser) return;

		window.removeEventListener('resize', onWindowResize);
		if (animationId) {
			cancelAnimationFrame(animationId);
		}
		if (renderer) {
			renderer.dispose();
		}
	});

	function init() {
		if (!container || !THREE || !OrbitControls) return;

		// 创建场景
		scene = new THREE.Scene();
		scene.background = new THREE.Color(colors.sky);
		scene.fog = new THREE.Fog(colors.sky, 20, 50);

		// 创建相机
		camera = new THREE.PerspectiveCamera(
			50,
			container.clientWidth / container.clientHeight,
			0.1,
			1000
		);
		camera.position.set(8, 6, 10);

		// 创建渲染器
		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(container.clientWidth, container.clientHeight);
		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;
		container.appendChild(renderer.domElement);

		// 创建轨道控制器
		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.minDistance = 5;
		controls.maxDistance = 30;
		controls.maxPolarAngle = Math.PI / 2 + 0.1;

		// 添加灯光
		addLights();

		// 创建日晷
		createSundial();

		// 添加地面
		addGround();
	}

	function addLights() {
		if (!scene || !THREE) return;

		// 环境光
		const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
		scene.add(ambientLight);

		// 主光源（模拟太阳光）
		const sunLight = new THREE.DirectionalLight(colors.sunlight, 1.5);
		sunLight.position.set(10, 15, 5);
		sunLight.castShadow = true;
		sunLight.shadow.mapSize.width = 2048;
		sunLight.shadow.mapSize.height = 2048;
		sunLight.shadow.camera.near = 0.5;
		sunLight.shadow.camera.far = 50;
		sunLight.shadow.camera.left = -15;
		sunLight.shadow.camera.right = 15;
		sunLight.shadow.camera.top = 15;
		sunLight.shadow.camera.bottom = -15;
		scene.add(sunLight);

		// 辅助光源
		const fillLight = new THREE.DirectionalLight(0xffffff, 0.3);
		fillLight.position.set(-10, 5, -5);
		scene.add(fillLight);
	}

	function createSundial() {
		if (!scene || !THREE) return;

		sundialGroup = new THREE.Group();

		// 创建底座
		createBase();

		// 创建赤道盘
		createEquatorialDisk();

		// 创建指针（晷针）
		createGnomon();

		// 创建小时标记
		createHourMarkers();

		// 调整日晷的角度，根据纬度倾斜
		sundialGroup.rotation.x = (latitude * Math.PI) / 180;

		scene.add(sundialGroup);
	}

	function createBase() {
		if (!sundialGroup || !THREE) return;

		// 底座主体
		const baseGeometry = new THREE.CylinderGeometry(3, 3.5, 0.8, 64);
		const baseMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.8,
			metalness: 0.2
		});
		const base = new THREE.Mesh(baseGeometry, baseMaterial);
		base.position.y = 0.4;
		base.castShadow = true;
		base.receiveShadow = true;
		sundialGroup.add(base);

		// 底座顶部平台
		const platformGeometry = new THREE.CylinderGeometry(2.5, 3, 0.3, 64);
		const platformMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.7,
			metalness: 0.1
		});
		const platform = new THREE.Mesh(platformGeometry, platformMaterial);
		platform.position.y = 1.0;
		platform.castShadow = true;
		platform.receiveShadow = true;
		sundialGroup.add(platform);

		// 支柱
		const pillarGeometry = new THREE.CylinderGeometry(0.3, 0.5, 2, 32);
		const pillarMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.6,
			metalness: 0.1
		});
		const pillar = new THREE.Mesh(pillarGeometry, pillarMaterial);
		pillar.position.y = 2.0;
		pillar.castShadow = true;
		pillar.receiveShadow = true;
		sundialGroup.add(pillar);
	}

	function createEquatorialDisk() {
		if (!sundialGroup || !THREE) return;

		// 赤道盘主体
		const diskGeometry = new THREE.CylinderGeometry(2.8, 2.8, 0.1, 128);
		const diskMaterial = new THREE.MeshStandardMaterial({
			color: colors.disk,
			roughness: 0.9,
			metalness: 0.1
		});
		const equatorialDisk = new THREE.Mesh(diskGeometry, diskMaterial);
		equatorialDisk.position.y = 3.2;
		equatorialDisk.castShadow = true;
		equatorialDisk.receiveShadow = true;
		sundialGroup.add(equatorialDisk);

		// 赤道盘边框
		const ringGeometry = new THREE.TorusGeometry(2.85, 0.08, 16, 128);
		const ringMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.5,
			metalness: 0.2
		});
		const ring = new THREE.Mesh(ringGeometry, ringMaterial);
		ring.position.y = 3.2;
		ring.rotation.x = Math.PI / 2;
		ring.castShadow = true;
		sundialGroup.add(ring);
	}

	function createGnomon() {
		if (!sundialGroup || !THREE) return;

		// 晷针主体（三角形）
		const gnomonHeight = 2.5;
		const gnomonWidth = 0.05;
		const gnomonDepth = 0.4;

		// 创建一个三角形棱柱作为晷针
		const shape = new THREE.Shape();
		shape.moveTo(0, 0);
		shape.lineTo(gnomonDepth, 0);
		shape.lineTo(gnomonDepth / 2, gnomonHeight);
		shape.closePath();

		const extrudeSettings = {
			steps: 1,
			depth: gnomonWidth,
			bevelEnabled: false
		};

		const gnomonGeometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
		const gnomonMaterial = new THREE.MeshStandardMaterial({
			color: colors.gnomon,
			roughness: 0.3,
			metalness: 0.7
		});
		const gnomon = new THREE.Mesh(gnomonGeometry, gnomonMaterial);

		// 定位晷针
		gnomon.position.y = 3.25;
		gnomon.position.z = -gnomonDepth / 2;
		gnomon.rotation.x = Math.PI / 2;
		gnomon.castShadow = true;
		gnomon.receiveShadow = true;
		sundialGroup.add(gnomon);

		// 晷针底座
		const baseGeometry = new THREE.CylinderGeometry(0.1, 0.15, 0.1, 32);
		const baseMaterial = new THREE.MeshStandardMaterial({
			color: colors.gnomon,
			roughness: 0.3,
			metalness: 0.7
		});
		const gnomonBase = new THREE.Mesh(baseGeometry, baseMaterial);
		gnomonBase.position.y = 3.25;
		gnomonBase.castShadow = true;
		gnomonBase.receiveShadow = true;
		sundialGroup.add(gnomonBase);
	}

	function createHourMarkers() {
		if (!sundialGroup || !THREE) return;

		const hourMarkers = new THREE.Group();
		hourMarkers.position.y = 3.26;

		const markerInnerRadius = 2.2;
		const markerOuterRadius = 2.6;

		// 创建12个小时标记（从6点到18点）
		for (let i = 0; i < 12; i++) {
			const hour = 6 + i; // 6:00 到 18:00
			const angle = (i * 30 * Math.PI) / 180; // 每小时30度

			// 主刻度线
			const isMainHour = hour % 3 === 0;
			const lineLength = isMainHour ? 0.4 : 0.25;
			const lineWidth = isMainHour ? 0.06 : 0.03;

			const lineGeometry = new THREE.BoxGeometry(lineWidth, 0.02, lineLength);
			const lineMaterial = new THREE.MeshStandardMaterial({
				color: colors.hourMarker
			});
			const line = new THREE.Mesh(lineGeometry, lineMaterial);

			// 定位刻度线
			const radius = (markerInnerRadius + markerOuterRadius) / 2;
			line.position.x = Math.cos(angle) * radius;
			line.position.z = Math.sin(angle) * radius;
			line.rotation.y = -angle;

			hourMarkers.add(line);

			// 小时数字（仅主小时）
			if (isMainHour) {
				const hourText = createTextSprite(hour.toString());
				const textRadius = markerInnerRadius - 0.3;
				hourText.position.x = Math.cos(angle) * textRadius;
				hourText.position.z = Math.sin(angle) * textRadius;
				hourText.position.y = 0.05;
				hourMarkers.add(hourText);
			}
		}

		// 添加分钟刻度
		for (let i = 0; i < 60; i++) {
			if (i % 5 === 0) continue; // 跳过整点刻度

			const angle = (i * 6 * Math.PI) / 180; // 每分钟6度
			const lineLength = 0.1;
			const lineWidth = 0.02;

			const lineGeometry = new THREE.BoxGeometry(lineWidth, 0.02, lineLength);
			const lineMaterial = new THREE.MeshStandardMaterial({
				color: colors.hourMarker
			});
			const line = new THREE.Mesh(lineGeometry, lineMaterial);

			const radius = (markerInnerRadius + markerOuterRadius) / 2;
			line.position.x = Math.cos(angle) * radius;
			line.position.z = Math.sin(angle) * radius;
			line.rotation.y = -angle;

			hourMarkers.add(line);
		}

		sundialGroup.add(hourMarkers);
	}

	function createTextSprite(text: string) {
		if (!THREE) return null;

		const canvas = document.createElement('canvas');
		const context = canvas.getContext('2d')!;
		canvas.width = 128;
		canvas.height = 128;

		context.font = 'Bold 80px Arial';
		context.fillStyle = '#000000';
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillText(text, 64, 64);

		const texture = new THREE.CanvasTexture(canvas);
		const material = new THREE.SpriteMaterial({ map: texture });
		const sprite = new THREE.Sprite(material);
		sprite.scale.set(0.3, 0.3, 1);
		sprite.rotation.x = -Math.PI / 2;

		return sprite;
	}

	function addGround() {
		if (!scene || !THREE) return;

		const groundGeometry = new THREE.PlaneGeometry(50, 50);
		const groundMaterial = new THREE.MeshStandardMaterial({
			color: 0x228B22,
			roughness: 0.9,
			metalness: 0.1
		});
		const ground = new THREE.Mesh(groundGeometry, groundMaterial);
		ground.rotation.x = -Math.PI / 2;
		ground.position.y = -0.01;
		ground.receiveShadow = true;
		scene.add(ground);
	}

	function calculateSolarTime(): number {
		const now = new Date();
		
		// 计算真太阳时
		const dayOfYear = getDayOfYear(now);
		const equationOfTime = calculateEquationOfTime(dayOfYear);
		
		// 计算地方时
		const utcHours = now.getUTCHours() + now.getUTCMinutes() / 60 + now.getUTCSeconds() / 3600;
		const localStandardTime = utcHours + longitude / 15;
		
		// 真太阳时 = 地方时 + 均时差
		let solarTime = localStandardTime + equationOfTime / 60;
		
		// 归一化到0-24小时
		while (solarTime < 0) solarTime += 24;
		while (solarTime >= 24) solarTime -= 24;
		
		return solarTime;
	}

	function getDayOfYear(date: Date): number {
		const start = new Date(date.getFullYear(), 0, 0);
		const diff = date.getTime() - start.getTime();
		const oneDay = 1000 * 60 * 60 * 24;
		return Math.floor(diff / oneDay);
	}

	function calculateEquationOfTime(dayOfYear: number): number {
		// 简化的均时差计算（分钟）
		const b = (2 * Math.PI * (dayOfYear - 81)) / 364;
		return 9.87 * Math.sin(2 * b) - 7.53 * Math.cos(b) - 1.5 * Math.sin(b);
	}

	function updateSunPosition() {
		if (!scene || !THREE) return;

		const solarTime = calculateSolarTime();
		
		// 计算太阳位置（简化模型）
		// 中午12点太阳在正南，高度角根据纬度和季节变化
		const hourAngle = (solarTime - 12) * 15; // 每小时15度
		
		// 计算太阳赤纬角（简化）
		const now = new Date();
		const dayOfYear = getDayOfYear(now);
		const declination = 23.45 * Math.sin((2 * Math.PI * (dayOfYear - 81)) / 365);
		
		// 计算太阳高度角和方位角
		const latitudeRad = (latitude * Math.PI) / 180;
		const declinationRad = (declination * Math.PI) / 180;
		const hourAngleRad = (hourAngle * Math.PI) / 180;
		
		const sinAltitude = 
			Math.sin(latitudeRad) * Math.sin(declinationRad) +
			Math.cos(latitudeRad) * Math.cos(declinationRad) * Math.cos(hourAngleRad);
		const altitude = Math.asin(sinAltitude);
		
		const cosAzimuth = 
			(Math.sin(altitude) * Math.sin(latitudeRad) - Math.sin(declinationRad)) /
			(Math.cos(altitude) * Math.cos(latitudeRad));
		let azimuth = Math.acos(Math.max(-1, Math.min(1, cosAzimuth)));
		
		if (solarTime > 12) {
			azimuth = -azimuth;
		}
		
		// 更新太阳光位置
		const sunLight = scene.children.find((child: any) => child instanceof THREE.DirectionalLight && child.intensity === 1.5);
		if (sunLight) {
			const distance = 20;
			sunLight.position.x = distance * Math.sin(azimuth) * Math.cos(altitude);
			sunLight.position.y = distance * Math.sin(altitude);
			sunLight.position.z = distance * Math.cos(azimuth) * Math.cos(altitude);
		}
	}

	function animate() {
		if (!browser || !renderer || !scene || !camera || !controls) return;

		animationId = requestAnimationFrame(animate);
		
		// 更新太阳位置
		updateSunPosition();
		
		// 更新控制器
		controls.update();
		
		// 渲染
		renderer.render(scene, camera);
	}

	function onWindowResize() {
		if (!camera || !renderer || !container) return;

		camera.aspect = container.clientWidth / container.clientHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(container.clientWidth, container.clientHeight);
	}
</script>

<div class="sundial-container" bind:this={container}>
	<!-- 3D日晷将渲染在这里 -->
</div>

<style>
	.sundial-container {
		width: 100%;
		height: 100vh;
		position: relative;
		overflow: hidden;
	}

	.sundial-container :global(canvas) {
		display: block;
		width: 100%;
		height: 100%;
	}
</style>
