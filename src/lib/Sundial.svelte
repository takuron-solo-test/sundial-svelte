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
	let sunObject: any;
	let sunLight: any;
	let latitude: number = 39.9; // 北京纬度
	let longitude: number = 116.4; // 北京经度
	let currentTimeText: string = $state('');
	let currentSolarTimeText: string = $state('');

	// 颜色配置
	const colors = {
		base: 0x8b4513, // 棕色
		disk: 0xfff8dc, // 米白色
		gnomon: 0x2a2a2a, // 深灰色（代替纯黑，更有质感）
		hourMarker: 0x1a1a1a, // 深灰色刻度
		highlight: 0xff4444, // 红色（当前小时）
		sunlight: 0xffffcc, // 太阳光颜色
		sun: 0xffd700, // 太阳金色
		sunGlow: 0xffa500, // 太阳光晕橙色
		sky: 0x87ceeb, // 天空蓝色
		skyTop: 0x1e90ff, // 天空顶部深蓝色
		skyBottom: 0xb0e0e6 // 天空底部浅蓝色
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
		
		// 创建渐变天空背景
		createSkyBackground();

		// 创建相机 - 更大的视场角，让太阳更容易进入视野
		camera = new THREE.PerspectiveCamera(
			75,
			container.clientWidth / container.clientHeight,
			0.1,
			1000
		);
		// 调整相机位置，让太阳更可能在视野中
		camera.position.set(12, 10, 15);

		// 创建渲染器
		renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
		renderer.setSize(container.clientWidth, container.clientHeight);
		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;
		renderer.toneMapping = THREE.ACESFilmicToneMapping;
		renderer.toneMappingExposure = 1.2;
		container.appendChild(renderer.domElement);

		// 创建轨道控制器
		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.minDistance = 5;
		controls.maxDistance = 40;
		controls.maxPolarAngle = Math.PI / 2 + 0.2;
		controls.target.set(0, 3, 0);

		// 添加灯光
		addLights();

		// 创建可见的太阳
		createSun();

		// 创建日晷
		createSundial();

		// 添加地面
		addGround();

		// 初始更新时间
		updateTimeDisplay();
	}

	function createSkyBackground() {
		// 创建天空几何体
		const skyGeometry = new THREE.SphereGeometry(100, 32, 32);
		const skyMaterial = new THREE.ShaderMaterial({
			uniforms: {
				topColor: { value: new THREE.Color(colors.skyTop) },
				bottomColor: { value: new THREE.Color(colors.skyBottom) },
				offset: { value: 20 },
				exponent: { value: 0.6 }
			},
			vertexShader: `
				varying vec3 vWorldPosition;
				void main() {
					vec4 worldPosition = modelMatrix * vec4(position, 1.0);
					vWorldPosition = worldPosition.xyz;
					gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
				}
			`,
			fragmentShader: `
				uniform vec3 topColor;
				uniform vec3 bottomColor;
				uniform float offset;
				uniform float exponent;
				varying vec3 vWorldPosition;
				void main() {
					float h = normalize(vWorldPosition + offset).y;
					gl_FragColor = vec4(mix(bottomColor, topColor, max(pow(max(h, 0.0), exponent), 0.0)), 1.0);
				}
			`,
			side: THREE.BackSide
		});
		const sky = new THREE.Mesh(skyGeometry, skyMaterial);
		scene.add(sky);
	}

	function addLights() {
		// 环境光 - 稍弱一些，让阴影更明显
		const ambientLight = new THREE.AmbientLight(0x404040, 0.5);
		scene.add(ambientLight);

		// 主光源（模拟太阳光）- 稍后会在updateSunPosition中更新位置
		sunLight = new THREE.DirectionalLight(colors.sunlight, 2.5);
		sunLight.position.set(10, 15, 5);
		sunLight.castShadow = true;
		
		// 优化阴影设置 - 更自然的阴影
		sunLight.shadow.mapSize.width = 4096;
		sunLight.shadow.mapSize.height = 4096;
		sunLight.shadow.camera.near = 0.5;
		sunLight.shadow.camera.far = 100;
		sunLight.shadow.camera.left = -40;
		sunLight.shadow.camera.right = 40;
		sunLight.shadow.camera.top = 40;
		sunLight.shadow.camera.bottom = -40;
		sunLight.shadow.bias = -0.0005;
		sunLight.shadow.normalBias = 0.05;
		// 让阴影边缘更柔和
		sunLight.shadow.radius = 1.5;
		
		scene.add(sunLight);

		// 半球光，增强环境光效果
		const hemiLight = new THREE.HemisphereLight(0x87ceeb, 0x8b4513, 0.3);
		scene.add(hemiLight);

		// 辅助光源 - 稍弱一些
		const fillLight = new THREE.DirectionalLight(0xffffff, 0.2);
		fillLight.position.set(-10, 5, -5);
		scene.add(fillLight);
	}

	function createSun() {
		// 太阳主体 - 适中大小
		const sunGeometry = new THREE.SphereGeometry(2.0, 64, 64);
		const sunMaterial = new THREE.MeshBasicMaterial({
			color: colors.sun,
			transparent: false
		});
		sunObject = new THREE.Mesh(sunGeometry, sunMaterial);

		// 太阳光晕（多层同心球，从内向外，更自然的光晕）
		for (let i = 1; i <= 5; i++) {
			const glowGeometry = new THREE.SphereGeometry(2.0 + i * 0.8, 32, 32);
			const glowMaterial = new THREE.MeshBasicMaterial({
				color: i <= 2 ? colors.sunGlow : colors.sunlight,
				transparent: true,
				opacity: 0.5 / i,
				side: THREE.BackSide
			});
			const glow = new THREE.Mesh(glowGeometry, glowMaterial);
			sunObject.add(glow);
		}

		// 初始位置
		sunObject.position.set(12, 10, 6);
		scene.add(sunObject);
	}

	function createSundial() {
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
		// 赤道式日晷的盘面应平行于赤道面，与地平面的夹角为90°-纬度
		sundialGroup.rotation.x = ((90 - latitude) * Math.PI) / 180;

		scene.add(sundialGroup);
	}

	function createBase() {
		// 底座主体 - 多层圆柱形，更有层次感
		const baseGroup = new THREE.Group();

		// 最底层
		const base1Geometry = new THREE.CylinderGeometry(4, 4.5, 0.5, 64);
		const baseMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.85,
			metalness: 0.15
		});
		const base1 = new THREE.Mesh(base1Geometry, baseMaterial);
		base1.position.y = 0.25;
		base1.castShadow = true;
		base1.receiveShadow = true;
		baseGroup.add(base1);

		// 第二层
		const base2Geometry = new THREE.CylinderGeometry(3.2, 4, 0.5, 64);
		const base2 = new THREE.Mesh(base2Geometry, baseMaterial);
		base2.position.y = 0.75;
		base2.castShadow = true;
		base2.receiveShadow = true;
		baseGroup.add(base2);

		// 第三层（平台）
		const platformGeometry = new THREE.CylinderGeometry(2.8, 3.2, 0.4, 64);
		const platformMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.7,
			metalness: 0.2
		});
		const platform = new THREE.Mesh(platformGeometry, platformMaterial);
		platform.position.y = 1.2;
		platform.castShadow = true;
		platform.receiveShadow = true;
		baseGroup.add(platform);

		// 支柱
		const pillarGeometry = new THREE.CylinderGeometry(0.35, 0.55, 2.2, 32);
		const pillarMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.6,
			metalness: 0.25
		});
		const pillar = new THREE.Mesh(pillarGeometry, pillarMaterial);
		pillar.position.y = 2.5;
		pillar.castShadow = true;
		pillar.receiveShadow = true;
		baseGroup.add(pillar);

		// 支柱顶部装饰
		const capGeometry = new THREE.CylinderGeometry(0.8, 0.35, 0.2, 32);
		const capMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.5,
			metalness: 0.3
		});
		const cap = new THREE.Mesh(capGeometry, capMaterial);
		cap.position.y = 3.7;
		cap.castShadow = true;
		cap.receiveShadow = true;
		baseGroup.add(cap);

		sundialGroup.add(baseGroup);
	}

	function createEquatorialDisk() {
		// 赤道盘主体
		const diskGeometry = new THREE.CylinderGeometry(3.2, 3.2, 0.15, 128);
		const diskMaterial = new THREE.MeshStandardMaterial({
			color: colors.disk,
			roughness: 0.9,
			metalness: 0.05
		});
		const equatorialDisk = new THREE.Mesh(diskGeometry, diskMaterial);
		equatorialDisk.position.y = 4.0;
		equatorialDisk.castShadow = true;
		equatorialDisk.receiveShadow = true;
		sundialGroup.add(equatorialDisk);

		// 赤道盘边框 - 更精致的装饰
		const ringGeometry = new THREE.TorusGeometry(3.28, 0.12, 16, 128);
		const ringMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.4,
			metalness: 0.5
		});
		const ring = new THREE.Mesh(ringGeometry, ringMaterial);
		ring.position.y = 4.0;
		ring.rotation.x = Math.PI / 2;
		ring.castShadow = true;
		ring.receiveShadow = true;
		sundialGroup.add(ring);

		// 盘面内圈装饰线
		const innerRingGeometry = new THREE.TorusGeometry(1.0, 0.02, 8, 64);
		const innerRingMaterial = new THREE.MeshStandardMaterial({
			color: colors.hourMarker,
			roughness: 0.8,
			metalness: 0.1
		});
		const innerRing = new THREE.Mesh(innerRingGeometry, innerRingMaterial);
		innerRing.position.y = 4.08;
		innerRing.rotation.x = Math.PI / 2;
		sundialGroup.add(innerRing);
	}

	function createGnomon() {
		// 晷针（指针）- 赤道式日晷的指针应平行于地轴
		// 指针应指向北极星方向
		
		const gnomonGroup = new THREE.Group();
		
		// 指针主体 - 更粗更明显的三棱柱
		const gnomonHeight = 4.0; // 增加高度，使阴影更明显
		const gnomonWidth = 0.12; // 增加宽度，更明显
		const gnomonDepth = 0.7; // 增加深度

		// 创建一个更精致的三角形棱柱
		const shape = new THREE.Shape();
		shape.moveTo(0, 0);
		shape.lineTo(gnomonDepth, 0);
		shape.lineTo(gnomonDepth / 2, gnomonHeight);
		shape.closePath();

		const extrudeSettings = {
			steps: 1,
			depth: gnomonWidth,
			bevelEnabled: true,
			bevelThickness: 0.02,
			bevelSize: 0.02,
			bevelSegments: 2
		};

		const gnomonGeometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
		const gnomonMaterial = new THREE.MeshStandardMaterial({
			color: 0x1a1a1a, // 更深的黑色
			roughness: 0.3,
			metalness: 0.5
		});
		const gnomon = new THREE.Mesh(gnomonGeometry, gnomonMaterial);

		// 定位晷针 - 指针应垂直于赤道盘
		gnomon.position.y = 4.08;
		gnomon.position.z = -gnomonDepth / 2;
		gnomon.rotation.x = Math.PI / 2;
		gnomon.castShadow = true;
		gnomon.receiveShadow = true;
		gnomonGroup.add(gnomon);

		// 晷针底座装饰 - 更大更明显
		const baseGeometry = new THREE.CylinderGeometry(0.3, 0.4, 0.2, 32);
		const baseMaterial = new THREE.MeshStandardMaterial({
			color: 0x2a2a2a,
			roughness: 0.3,
			metalness: 0.6
		});
		const gnomonBase = new THREE.Mesh(baseGeometry, baseMaterial);
		gnomonBase.position.y = 4.15;
		gnomonBase.castShadow = true;
		gnomonBase.receiveShadow = true;
		gnomonGroup.add(gnomonBase);

		// 指针顶部装饰 - 更大更明显的金色顶部
		const tipGeometry = new THREE.ConeGeometry(0.1, 0.25, 16);
		const tipMaterial = new THREE.MeshStandardMaterial({
			color: 0xffd700, // 金色顶部
			roughness: 0.2,
			metalness: 0.9
		});
		const tip = new THREE.Mesh(tipGeometry, tipMaterial);
		tip.position.y = 4.08 + gnomonHeight;
		tip.position.z = -gnomonDepth / 2;
		tip.rotation.x = Math.PI / 2;
		tip.castShadow = true;
		gnomonGroup.add(tip);

		sundialGroup.add(gnomonGroup);
	}

	function createHourMarkers() {
		const hourMarkersGroup = new THREE.Group();
		hourMarkersGroup.position.y = 4.08;

		const markerInnerRadius = 2.4;
		const markerOuterRadius = 3.0;

		// 赤道式日晷的正确标记：
		// 12点在北(z负方向/上方)
		// 6点在南(z正方向/下方)
		// 3点在东(x正方向/右)
		// 9点在西(x负方向/左)
		// 每小时15度，从北开始顺时针旋转

		// 创建24个小时标记（从0点到23点，覆盖全天）
		for (let i = 0; i < 24; i++) {
			const hour = i;
			// 角度计算：从北(12点)开始，顺时针方向
			// 12点: 0度，3点: 90度(东)，6点: 180度(南)，9点: 270度(西)
			// 注意：赤道式日晷的小时标记是每小时15度
			const angle = ((hour - 12) * 15 * Math.PI) / 180;

			// 主刻度线
			const isMainHour = hour % 3 === 0;
			const isQuarterHour = hour % 6 === 0;
			let lineLength = isQuarterHour ? 0.5 : (isMainHour ? 0.35 : 0.2);
			let lineWidth = isQuarterHour ? 0.08 : (isMainHour ? 0.05 : 0.025);

			const lineGeometry = new THREE.BoxGeometry(lineWidth, 0.02, lineLength);
			const lineMaterial = new THREE.MeshStandardMaterial({
				color: colors.hourMarker
			});
			const line = new THREE.Mesh(lineGeometry, lineMaterial);

			// 定位刻度线 - 沿半径方向
			const radius = (markerInnerRadius + markerOuterRadius) / 2;
			line.position.x = Math.sin(angle) * radius;
			line.position.z = -Math.cos(angle) * radius;
			line.rotation.y = -angle;

			hourMarkersGroup.add(line);

			// 小时数字（仅主小时）
			if (isQuarterHour) {
				// 显示小时数字，明确区分0点和12点
				let displayHour = hour;
				// 0点显示"0"，12点显示"12"
				if (displayHour > 12) displayHour -= 12;
				
				const hourText = createTextSprite(displayHour.toString());
				const textRadius = markerInnerRadius - 0.5;
				hourText.position.x = Math.sin(angle) * textRadius;
				hourText.position.z = -Math.cos(angle) * textRadius;
				hourText.position.y = 0.05;
				
				// 调整文字朝向 - 让文字朝外
				hourText.rotation.x = -Math.PI / 2;
				hourText.rotation.z = angle;
				
				hourMarkersGroup.add(hourText);
			}
		}

		// 添加分钟刻度
		for (let i = 0; i < 60; i++) {
			if (i % 5 === 0) continue; // 跳过整点刻度

			const minuteAngle = (i * 6 * Math.PI) / 180; // 每分钟6度
			const lineLength = 0.1;
			const lineWidth = 0.015;

			const lineGeometry = new THREE.BoxGeometry(lineWidth, 0.015, lineLength);
			const lineMaterial = new THREE.MeshStandardMaterial({
				color: colors.hourMarker
			});
			const line = new THREE.Mesh(lineGeometry, lineMaterial);

			const radius = (markerInnerRadius + markerOuterRadius) / 2;
			line.position.x = Math.sin(minuteAngle) * radius;
			line.position.z = -Math.cos(minuteAngle) * radius;
			line.rotation.y = -minuteAngle;

			hourMarkersGroup.add(line);
		}

		sundialGroup.add(hourMarkersGroup);
	}

	function createTextSprite(text: string) {
		if (!THREE) return null;

		const canvas = document.createElement('canvas');
		const context = canvas.getContext('2d')!;
		canvas.width = 256;
		canvas.height = 256;

		// 绘制背景（透明）
		context.fillStyle = 'rgba(255, 255, 255, 0)';
		context.fillRect(0, 0, 256, 256);

		// 绘制文字阴影
		context.font = 'Bold 120px Arial';
		context.fillStyle = 'rgba(0, 0, 0, 0.3)';
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillText(text, 130, 134);

		// 绘制文字
		context.fillStyle = colors.hourMarker.toString(16);
		context.fillText(text, 128, 128);

		const texture = new THREE.CanvasTexture(canvas);
		texture.needsUpdate = true;
		const material = new THREE.SpriteMaterial({ 
			map: texture,
			transparent: true
		});
		const sprite = new THREE.Sprite(material);
		sprite.scale.set(0.4, 0.4, 1);
		sprite.rotation.x = -Math.PI / 2;

		return sprite;
	}

	function addGround() {
		// 地面 - 更自然的草地效果
		const groundGeometry = new THREE.PlaneGeometry(100, 100, 50, 50);
		
		// 简单的草地材质
		const groundMaterial = new THREE.MeshStandardMaterial({
			color: 0x2e8b57, // 深绿色草地
			roughness: 0.95,
			metalness: 0.0
		});
		
		const ground = new THREE.Mesh(groundGeometry, groundMaterial);
		ground.rotation.x = -Math.PI / 2;
		ground.position.y = -0.01;
		ground.receiveShadow = true;
		scene.add(ground);

		// 添加一些装饰性的石头或植物（简单的几何体）
		for (let i = 0; i < 20; i++) {
			const stoneGeometry = new THREE.SphereGeometry(
				0.2 + Math.random() * 0.3,
				8,
				8
			);
			const stoneMaterial = new THREE.MeshStandardMaterial({
				color: 0x808080 + Math.random() * 0x202020,
				roughness: 0.9,
				metalness: 0.1
			});
			const stone = new THREE.Mesh(stoneGeometry, stoneMaterial);
			
			// 随机分布在日晷周围
			const angle = Math.random() * Math.PI * 2;
			const distance = 8 + Math.random() * 15;
			stone.position.x = Math.cos(angle) * distance;
			stone.position.z = Math.sin(angle) * distance;
			stone.position.y = 0.1 + Math.random() * 0.2;
			
			stone.castShadow = true;
			stone.receiveShadow = true;
			scene.add(stone);
		}
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
		if (!scene || !THREE || !sunLight || !sunObject) return;

		const solarTime = calculateSolarTime();
		
		// 计算太阳位置（更精确的模型）
		// 中午12点太阳在正南，高度角根据纬度和季节变化
		const hourAngle = (solarTime - 12) * 15; // 每小时15度
		
		// 计算太阳赤纬角（更精确）
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
		const altitude = Math.asin(Math.max(-1, Math.min(1, sinAltitude)));
		
		// 计算方位角（从南方向东为正，顺时针）
		const cosAzimuth = 
			(Math.sin(altitude) * Math.sin(latitudeRad) - Math.sin(declinationRad)) /
			(Math.cos(altitude) * Math.cos(latitudeRad));
		let azimuth = Math.acos(Math.max(-1, Math.min(1, cosAzimuth)));
		
		// 上午（太阳时<12）太阳在东，方位角为正；下午在西，方位角为负
		// 注意：这里的时角是负的（上午）和正的（下午）
		if (hourAngle > 0) {
			azimuth = -azimuth;
		}
		
		// 调试：打印太阳位置
		// console.log('Solar Time:', solarTime.toFixed(2), 'Hour Angle:', hourAngle.toFixed(1), 
		//             'Altitude:', (altitude * 180/Math.PI).toFixed(1), 
		//             'Azimuth:', (azimuth * 180/Math.PI).toFixed(1),
		//             'Declination:', declination.toFixed(1));
		
		// 使用更简单直观的方式计算太阳位置
		// 坐标系：x=东, y=上, z=北（z负方向是北，z正方向是南）
		// 方位角从南方向东为正（顺时针）
		
		// 转换为坐标：
		// 南方向是z正方向，东方向是x正方向
		// 当方位角为正时（向东），x增加，z减少（从南向东北转）
		// 当方位角为负时（向西），x减少，z减少（从南向西转）
		
		// 重新计算：方位角从南开始，向东为正，向西为负
		// 南：z正方向
		// 东：x正方向
		// 北：z负方向
		// 西：x负方向
		
		// 简化坐标计算，确保太阳位置正确
		// 时角：(solarTime - 12) * 15度
		// 上午（时角<0）：太阳在东南方向
		// 中午（时角=0）：太阳在正南
		// 下午（时角>0）：太阳在西南方向
		
		// 更简单的方法：直接根据时角计算水平方向
		// 时角=0（中午）：正南（z正方向）
		// 时角<0（上午）：向东转，x增加，z减少
		// 时角>0（下午）：向西转，x减少，z减少
		
		// 计算水平方向的分量（基于时角）
		// 时角每小时15度，从中午开始计算
		const horizontalAngle = -hourAngleRad; // 转换：上午时角负，向东转
		
		// 计算太阳位置坐标
		// 使用适中的距离，确保太阳在视野中且自然
		const sunDistance = 18;
		
		// 水平分量：基于方位角
		// 方位角从南方向东为正
		// 南：z正方向
		// 东：x正方向
		// 所以：
		// x = sin(azimuth) * cos(altitude)  （东分量）
		// z = cos(azimuth) * cos(altitude)   （南分量，z正方向是南）
		
		// 但是等一下，让我重新验证坐标系
		// 在Three.js中：
		// - z轴正方向指向"后方"
		// - z轴负方向指向"前方"
		// 我们约定：
		// - 北 = z负方向
		// - 南 = z正方向
		// - 东 = x正方向
		// - 西 = x负方向
		
		// 方位角从南开始计算：
		// 正南：azimuth = 0, x=0, z=+1
		// 正东：azimuth = 90°, x=+1, z=0
		// 正北：azimuth = 180°或-180°, x=0, z=-1
		// 正西：azimuth = -90°或270°, x=-1, z=0
		
		// 所以坐标应该是：
		// x = sin(azimuth)  （东分量）
		// z = cos(azimuth)  （南分量，z正方向是南）
		
		// 但是，当azimuth从0增加到180时：
		// sin(azimuth)从0→1→0（东→西？不，sin(180)=0，sin(90)=1，sin(-90)=-1）
		// cos(azimuth)从1→0→-1（南→北）
		
		// 等等，这不对。让我重新思考：
		// 如果azimuth是从南方向东为正（顺时针旋转）：
		// - 正南：0度
		// - 正东：90度
		// - 正北：180度
		// - 正西：270度（或-90度）
		
		// 那么坐标应该是：
		// x = sin(azimuth)  （东分量）
		// z = -cos(azimuth) （注意负号！因为：
		//   - 南（0度）：-cos(0) = -1？不对，南应该是z正方向）
		
		// 我发现问题了！让我重新定义：
		// 如果z正方向是南，z负方向是北
		// 那么：
		// - 正南：z = +1
		// - 正北：z = -1
		// - 正东：x = +1
		// - 正西：x = -1
		
		// 方位角从南开始，向东为正：
		// 正南（0°）：x=0, z=+1
		// 正东（90°）：x=+1, z=0
		// 正北（180°）：x=0, z=-1
		// 正西（-90°或270°）：x=-1, z=0
		
		// 这个模式符合：
		// x = sin(azimuth)
		// z = cos(azimuth)
		// 验证：
		// 0°: sin=0, cos=1 → x=0, z=+1 ✓（正南）
		// 90°: sin=1, cos=0 → x=+1, z=0 ✓（正东）
		// 180°: sin=0, cos=-1 → x=0, z=-1 ✓（正北）
		// -90°: sin=-1, cos=0 → x=-1, z=0 ✓（正西）
		
		// 完美！所以坐标公式是：
		// x = sin(azimuth) * cos(altitude)
		// y = sin(altitude)
		// z = cos(azimuth) * cos(altitude)
		
		// 现在让我验证当前时间的太阳位置：
		// 当前时间：约10:00（真太阳时）
		// 时角 = (10 - 12) * 15 = -30度
		// 方位角应该是正的（向东），因为是上午
		// 假设azimuth ≈ 45度（东南方向）
		// x = sin(45°) * cos(高度角) ≈ 0.707 * cos(50°) ≈ 0.45
		// z = cos(45°) * cos(高度角) ≈ 0.707 * cos(50°) ≈ 0.45
		// 所以位置应该是(0.45 * 20, sin(50°)*20, 0.45 * 20) ≈ (9, 15, 9)
		// 这在相机视野中应该是可见的！
		
		// 计算最终坐标
		const x = sunDistance * Math.sin(azimuth) * Math.cos(altitude);
		const y = sunDistance * Math.sin(altitude);
		const z = sunDistance * Math.cos(azimuth) * Math.cos(altitude);
		
		// 更新太阳光位置
		sunLight.position.set(x, y, z);
		sunLight.target.position.set(0, 3, 0);
		sunLight.target.updateMatrixWorld();

		// 更新可见太阳位置
		if (sunObject) {
			sunObject.position.set(x, y, z);
			
			// 让太阳始终朝向相机
			if (camera) {
				sunObject.lookAt(camera.position);
			}
		}

		// 根据太阳高度调整光强
		const altitudeDegrees = (altitude * 180) / Math.PI;
		if (altitudeDegrees > 0) {
			// 白天 - 增强光强，使阴影更明显
			sunLight.intensity = 3.5 * Math.max(0.5, Math.sin(altitude));
			if (sunObject) sunObject.visible = true;
		} else {
			// 夜晚
			sunLight.intensity = 0.1;
			if (sunObject) sunObject.visible = false;
		}
	}

	function updateTimeDisplay() {
		const now = new Date();
		const solarTime = calculateSolarTime();
		
		// 格式化当前时间
		const hours = String(now.getHours()).padStart(2, '0');
		const minutes = String(now.getMinutes()).padStart(2, '0');
		const seconds = String(now.getSeconds()).padStart(2, '0');
		currentTimeText = `当前时间: ${hours}:${minutes}:${seconds}`;
		
		// 格式化真太阳时
		const solarHours = Math.floor(solarTime);
		const solarMinutes = Math.floor((solarTime - solarHours) * 60);
		const solarSeconds = Math.floor(((solarTime - solarHours) * 60 - solarMinutes) * 60);
		currentSolarTimeText = `真太阳时: ${String(solarHours).padStart(2, '0')}:${String(solarMinutes).padStart(2, '0')}:${String(solarSeconds).padStart(2, '0')}`;
	}

	function animate() {
		if (!browser || !renderer || !scene || !camera || !controls) return;

		animationId = requestAnimationFrame(animate);
		
		// 更新太阳位置
		updateSunPosition();
		
		// 更新时间显示
		updateTimeDisplay();
		
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
	<!-- 时间显示面板 -->
	<div class="time-panel">
		<div class="time-display">{currentTimeText}</div>
		<div class="solar-time-display">{currentSolarTimeText}</div>
		<div class="location-info">位置: 北京 (纬度: {latitude}°, 经度: {longitude}°)</div>
	</div>
	
	<!-- 操作提示 -->
	<div class="controls-hint">
		<div class="hint-item">🖱️ 拖拽旋转</div>
		<div class="hint-item">🔍 滚轮缩放</div>
		<div class="hint-item">⌨️ 右键平移</div>
	</div>
</div>

<style>
	.sundial-container {
		width: 100%;
		height: 100vh;
		position: relative;
		overflow: hidden;
		background: linear-gradient(to bottom, #87ceeb, #b0e0e6);
	}

	.sundial-container :global(canvas) {
		display: block;
		width: 100%;
		height: 100%;
	}

	/* 时间显示面板 */
	.time-panel {
		position: absolute;
		top: 20px;
		left: 20px;
		background: rgba(0, 0, 0, 0.7);
		color: white;
		padding: 15px 20px;
		border-radius: 10px;
		font-family: 'Arial', sans-serif;
		backdrop-filter: blur(10px);
		border: 1px solid rgba(255, 255, 255, 0.1);
		box-shadow: 0 4px 15px rgba(0, 0, 0, 0.3);
		z-index: 100;
	}

	.time-display {
		font-size: 18px;
		font-weight: bold;
		margin-bottom: 8px;
		color: #4ade80;
	}

	.solar-time-display {
		font-size: 16px;
		margin-bottom: 10px;
		color: #fbbf24;
	}

	.location-info {
		font-size: 12px;
		color: #d1d5db;
		opacity: 0.8;
	}

	/* 操作提示 */
	.controls-hint {
		position: absolute;
		bottom: 20px;
		right: 20px;
		background: rgba(0, 0, 0, 0.6);
		color: white;
		padding: 10px 15px;
		border-radius: 8px;
		font-family: 'Arial', sans-serif;
		font-size: 12px;
		backdrop-filter: blur(5px);
		border: 1px solid rgba(255, 255, 255, 0.1);
		display: flex;
		gap: 15px;
		z-index: 100;
	}

	.hint-item {
		display: flex;
		align-items: center;
		gap: 5px;
		opacity: 0.8;
	}
</style>
