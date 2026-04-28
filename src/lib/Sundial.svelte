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
	let sunLight: any;
	let skyMesh: any;
	let latitude: number = 39.9;
	let longitude: number = 116.4;
	let currentTimeText: string = $state('');
	let currentSolarTimeText: string = $state('');

	const colors = {
		base: 0x8b4513,
		disk: 0xfff8dc,
		gnomon: 0x1a1a1a,
		hourMarker: 0x1a1a1a,
		highlight: 0xff4444,
		sunlight: 0xffffcc
	};

	let THREE: any;
	let OrbitControls: any;

	onMount(async () => {
		if (!browser) return;

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

		scene = new THREE.Scene();

		camera = new THREE.PerspectiveCamera(
			60,
			container.clientWidth / container.clientHeight,
			0.1,
			1000
		);
		camera.position.set(8, 6, 10);

		renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
		renderer.setSize(container.clientWidth, container.clientHeight);
		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = THREE.PCFShadowMap;
		renderer.toneMapping = THREE.ACESFilmicToneMapping;
		renderer.toneMappingExposure = 1.0;
		container.appendChild(renderer.domElement);

		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.minDistance = 5;
		controls.maxDistance = 30;
		controls.maxPolarAngle = Math.PI / 2 + 0.2;
		controls.target.set(0, 3, 0);

		createSky();
		addLights();
		createSundial();
		addGround();

		updateTimeDisplay();
	}

	function createSky() {
		const skyGeometry = new THREE.SphereGeometry(500, 32, 32);
		const skyMaterial = new THREE.ShaderMaterial({
			uniforms: {
				topColor: { value: new THREE.Color(0x1e90ff) },
				bottomColor: { value: new THREE.Color(0xb0e0e6) },
				offset: { value: 20 },
				exponent: { value: 0.6 },
				sunPos: { value: new THREE.Vector3(0, 1, 0) },
				sunSize: { value: 0.05 },
				sunIntensity: { value: 1.0 }
			},
			vertexShader: `
				varying vec3 vWorldPosition;
				varying vec3 vLocalPosition;
				void main() {
					vec4 worldPosition = modelMatrix * vec4(position, 1.0);
					vWorldPosition = worldPosition.xyz;
					vLocalPosition = position;
					gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
				}
			`,
			fragmentShader: `
				uniform vec3 topColor;
				uniform vec3 bottomColor;
				uniform float offset;
				uniform float exponent;
				uniform vec3 sunPos;
				uniform float sunSize;
				uniform float sunIntensity;
				varying vec3 vWorldPosition;
				varying vec3 vLocalPosition;
				
				void main() {
					float h = normalize(vWorldPosition + offset).y;
					vec3 skyColor = mix(bottomColor, topColor, max(pow(max(h, 0.0), exponent), 0.0));
					
					vec3 sunDir = normalize(sunPos);
					vec3 viewDir = normalize(vLocalPosition);
					float sunDot = dot(viewDir, sunDir);
					
					vec3 finalColor = skyColor;
					
					if (sunIntensity > 0.01) {
						float sunFactor = smoothstep(1.0 - sunSize, 1.0 - sunSize * 0.3, sunDot);
						
						vec3 sunCoreColor = vec3(1.0, 1.0, 0.9) * sunIntensity;
						vec3 sunGlowColor = vec3(1.0, 0.8, 0.3) * sunIntensity;
						vec3 sunHaloColor = vec3(1.0, 0.6, 0.2) * sunIntensity;
						
						float glowFactor = smoothstep(0.7, 0.95, sunDot) * 0.6 * sunIntensity;
						float haloFactor = smoothstep(0.5, 0.85, sunDot) * 0.3 * sunIntensity;
						
						finalColor = mix(skyColor, sunHaloColor, haloFactor);
						finalColor = mix(finalColor, sunGlowColor, glowFactor);
						finalColor = mix(finalColor, sunCoreColor, sunFactor);
					}
					
					gl_FragColor = vec4(finalColor, 1.0);
				}
			`,
			side: THREE.BackSide
		});
		skyMesh = new THREE.Mesh(skyGeometry, skyMaterial);
		scene.add(skyMesh);
	}

	function addLights() {
		const ambientLight = new THREE.AmbientLight(0x505050, 0.4);
		scene.add(ambientLight);

		sunLight = new THREE.DirectionalLight(0xffffcc, 2.0);
		sunLight.position.set(10, 10, 5);
		sunLight.castShadow = true;
		
		sunLight.shadow.mapSize.width = 4096;
		sunLight.shadow.mapSize.height = 4096;
		sunLight.shadow.camera.near = 0.5;
		sunLight.shadow.camera.far = 100;
		sunLight.shadow.camera.left = -20;
		sunLight.shadow.camera.right = 20;
		sunLight.shadow.camera.top = 20;
		sunLight.shadow.camera.bottom = -20;
		sunLight.shadow.bias = -0.0005;
		sunLight.shadow.normalBias = 0.02;
		sunLight.shadow.radius = 1.0;
		
		scene.add(sunLight);

		const hemiLight = new THREE.HemisphereLight(0x87ceeb, 0x8b4513, 0.3);
		scene.add(hemiLight);
	}

	function createSundial() {
		sundialGroup = new THREE.Group();

		createBase();
		createEquatorialDisk();
		createGnomon();
		createHourMarkers();

		sundialGroup.rotation.x = ((90 - latitude) * Math.PI) / 180;

		scene.add(sundialGroup);
	}

	function createBase() {
		const baseGroup = new THREE.Group();

		const base1Geometry = new THREE.CylinderGeometry(3.5, 4, 0.4, 64);
		const baseMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.85,
			metalness: 0.15
		});
		const base1 = new THREE.Mesh(base1Geometry, baseMaterial);
		base1.position.y = 0.2;
		base1.castShadow = true;
		base1.receiveShadow = true;
		baseGroup.add(base1);

		const base2Geometry = new THREE.CylinderGeometry(2.8, 3.5, 0.4, 64);
		const base2 = new THREE.Mesh(base2Geometry, baseMaterial);
		base2.position.y = 0.6;
		base2.castShadow = true;
		base2.receiveShadow = true;
		baseGroup.add(base2);

		const pillarGeometry = new THREE.CylinderGeometry(0.3, 0.45, 2.0, 32);
		const pillar = new THREE.Mesh(pillarGeometry, baseMaterial);
		pillar.position.y = 1.8;
		pillar.castShadow = true;
		pillar.receiveShadow = true;
		baseGroup.add(pillar);

		sundialGroup.add(baseGroup);
	}

	function createEquatorialDisk() {
		const diskGeometry = new THREE.CylinderGeometry(3.0, 3.0, 0.12, 128);
		const diskMaterial = new THREE.MeshStandardMaterial({
			color: colors.disk,
			roughness: 0.9,
			metalness: 0.05
		});
		const equatorialDisk = new THREE.Mesh(diskGeometry, diskMaterial);
		equatorialDisk.position.y = 2.9;
		equatorialDisk.castShadow = true;
		equatorialDisk.receiveShadow = true;
		sundialGroup.add(equatorialDisk);

		const ringGeometry = new THREE.TorusGeometry(3.05, 0.1, 16, 128);
		const ringMaterial = new THREE.MeshStandardMaterial({
			color: colors.base,
			roughness: 0.4,
			metalness: 0.5
		});
		const ring = new THREE.Mesh(ringGeometry, ringMaterial);
		ring.position.y = 2.9;
		ring.rotation.x = Math.PI / 2;
		ring.castShadow = true;
		ring.receiveShadow = true;
		sundialGroup.add(ring);
	}

	function createGnomon() {
		const gnomonGroup = new THREE.Group();

		const gnomonHeight = 4.0;
		const gnomonTopRadius = 0.02;
		const gnomonBottomRadius = 0.15;

		const gnomonGeometry = new THREE.ConeGeometry(
			gnomonBottomRadius,
			gnomonHeight,
			32
		);

		const gnomonMaterial = new THREE.MeshStandardMaterial({
			color: 0x1a1a1a,
			roughness: 0.3,
			metalness: 0.5
		});
		const gnomon = new THREE.Mesh(gnomonGeometry, gnomonMaterial);

		gnomon.position.y = 2.96 + gnomonHeight / 2;
		gnomon.rotation.x = Math.PI;
		gnomon.castShadow = true;
		gnomon.receiveShadow = true;
		gnomonGroup.add(gnomon);

		const tipGeometry = new THREE.ConeGeometry(0.06, 0.3, 32);
		const tipMaterial = new THREE.MeshStandardMaterial({
			color: 0xffd700,
			roughness: 0.2,
			metalness: 0.8
		});
		const tip = new THREE.Mesh(tipGeometry, tipMaterial);
		tip.position.y = 2.96 + gnomonHeight + 0.15;
		tip.rotation.x = Math.PI;
		tip.castShadow = true;
		gnomonGroup.add(tip);

		sundialGroup.add(gnomonGroup);
	}

	function createHourMarkers() {
		const hourMarkersGroup = new THREE.Group();
		hourMarkersGroup.position.y = 2.96;

		const markerInnerRadius = 2.2;
		const markerOuterRadius = 2.8;

		for (let i = 0; i < 24; i++) {
			const hour = i;
			const angle = ((hour - 12) * 15 * Math.PI) / 180;

			const isNoon = hour === 12;
			const isMidnight = hour === 0;
			const isMainHour = hour % 3 === 0;
			
			const isNightHour = hour >= 20 || hour <= 4;

			if (isNightHour) {
				continue;
			}

			let lineLength = isNoon ? 0.6 : (isMainHour ? 0.45 : 0.25);
			let lineWidth = isNoon ? 0.08 : (isMainHour ? 0.05 : 0.025);

			const lineGeometry = new THREE.BoxGeometry(lineWidth, 0.02, lineLength);
			const lineMaterial = new THREE.MeshStandardMaterial({
				color: colors.hourMarker
			});
			const line = new THREE.Mesh(lineGeometry, lineMaterial);

			const radius = (markerInnerRadius + markerOuterRadius) / 2;
			line.position.x = Math.sin(angle) * radius;
			line.position.z = Math.cos(angle) * radius;
			line.rotation.y = -angle;

			hourMarkersGroup.add(line);

			if (isMainHour && !isMidnight) {
				let displayHour = hour;
				if (displayHour > 12) displayHour -= 12;
				
				const hourText = createTextSprite(displayHour.toString());
				if (hourText) {
					const textRadius = markerInnerRadius - 0.35;
					hourText.position.x = Math.sin(angle) * textRadius;
					hourText.position.z = Math.cos(angle) * textRadius;
					hourText.position.y = 0.05;
					
					hourText.rotation.x = -Math.PI / 2;
					hourText.rotation.z = angle;
					
					hourMarkersGroup.add(hourText);
				}
			}
		}

		sundialGroup.add(hourMarkersGroup);
	}

	function createTextSprite(text: string) {
		if (!THREE) return null;

		const canvas = document.createElement('canvas');
		const context = canvas.getContext('2d')!;
		canvas.width = 256;
		canvas.height = 256;

		context.fillStyle = 'rgba(255, 255, 255, 0)';
		context.fillRect(0, 0, 256, 256);

		context.font = 'Bold 100px Arial';
		context.fillStyle = 'rgba(0, 0, 0, 0.3)';
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillText(text, 130, 134);

		context.fillStyle = colors.hourMarker.toString(16);
		context.fillText(text, 128, 128);

		const texture = new THREE.CanvasTexture(canvas);
		texture.needsUpdate = true;
		const material = new THREE.SpriteMaterial({ 
			map: texture,
			transparent: true
		});
		const sprite = new THREE.Sprite(material);
		sprite.scale.set(0.35, 0.35, 1);

		return sprite;
	}

	function addGround() {
		const groundGeometry = new THREE.PlaneGeometry(100, 100, 50, 50);
		
		const groundMaterial = new THREE.MeshStandardMaterial({
			color: 0x2e8b57,
			roughness: 0.95,
			metalness: 0.0
		});
		
		const ground = new THREE.Mesh(groundGeometry, groundMaterial);
		ground.rotation.x = -Math.PI / 2;
		ground.position.y = -0.01;
		ground.receiveShadow = true;
		scene.add(ground);
	}

	function calculateSolarTime(): number {
		const now = new Date();
		
		const dayOfYear = getDayOfYear(now);
		const equationOfTime = calculateEquationOfTime(dayOfYear);
		
		const utcHours = now.getUTCHours() + now.getUTCMinutes() / 60 + now.getUTCSeconds() / 3600;
		const localStandardTime = utcHours + longitude / 15;
		
		let solarTime = localStandardTime + equationOfTime / 60;
		
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
		const b = (2 * Math.PI * (dayOfYear - 81)) / 364;
		return 9.87 * Math.sin(2 * b) - 7.53 * Math.cos(b) - 1.5 * Math.sin(b);
	}

	function updateSunAndSky() {
		if (!scene || !THREE || !sunLight) return;

		const solarTime = calculateSolarTime();
		const hourAngle = (solarTime - 12) * 15;
		
		const now = new Date();
		const dayOfYear = getDayOfYear(now);
		const declination = 23.45 * Math.sin((2 * Math.PI * (dayOfYear - 81)) / 365);
		
		const latitudeRad = (latitude * Math.PI) / 180;
		const declinationRad = (declination * Math.PI) / 180;
		const hourAngleRad = (hourAngle * Math.PI) / 180;
		
		const sinAltitude = 
			Math.sin(latitudeRad) * Math.sin(declinationRad) +
			Math.cos(latitudeRad) * Math.cos(declinationRad) * Math.cos(hourAngleRad);
		const altitude = Math.asin(Math.max(-1, Math.min(1, sinAltitude)));
		
		const cosAzimuth = 
			(Math.sin(altitude) * Math.sin(latitudeRad) - Math.sin(declinationRad)) /
			(Math.cos(altitude) * Math.cos(latitudeRad));
		let azimuth = Math.acos(Math.max(-1, Math.min(1, cosAzimuth)));
		
		if (hourAngle < 0) {
			azimuth = -azimuth;
		}

		const sunDistance = 100;
		const sunX = -sunDistance * Math.sin(azimuth) * Math.cos(altitude);
		const sunY = sunDistance * Math.sin(altitude);
		const sunZ = -sunDistance * Math.cos(azimuth) * Math.cos(altitude);

		sunLight.position.set(sunX, sunY, sunZ);
		sunLight.target.position.set(0, 3, 0);
		sunLight.target.updateMatrixWorld();

		if (skyMesh && skyMesh.material.uniforms) {
			const sunDir = new THREE.Vector3(sunX, sunY, sunZ).normalize();
			skyMesh.material.uniforms.sunPos.value = sunDir;
			
			const altitudeDegrees = (altitude * 180) / Math.PI;
			let sunIntensity = 1.0;
			let topColor = new THREE.Color(0x1e90ff);
			let bottomColor = new THREE.Color(0xb0e0e6);
			
			if (altitudeDegrees < -6) {
				sunIntensity = 0.0;
				topColor = new THREE.Color(0x0a0a20);
				bottomColor = new THREE.Color(0x1a1a3a);
				sunLight.intensity = 0.1;
			} else if (altitudeDegrees < 0) {
				const t = (altitudeDegrees + 6) / 6;
				sunIntensity = t;
				topColor = new THREE.Color().lerpColors(
					new THREE.Color(0x0a0a20),
					new THREE.Color(0xff6b35),
					t
				);
				bottomColor = new THREE.Color().lerpColors(
					new THREE.Color(0x1a1a3a),
					new THREE.Color(0xffd4a3),
					t
				);
				sunLight.intensity = 0.5 + t * 1.5;
			} else if (altitudeDegrees < 15) {
				const t = altitudeDegrees / 15;
				sunIntensity = 0.8 + t * 0.2;
				topColor = new THREE.Color().lerpColors(
					new THREE.Color(0xff6b35),
					new THREE.Color(0x1e90ff),
					t
				);
				bottomColor = new THREE.Color().lerpColors(
					new THREE.Color(0xffd4a3),
					new THREE.Color(0xb0e0e6),
					t
				);
				sunLight.intensity = 1.5 + t * 1.0;
			} else {
				sunIntensity = 1.0;
				topColor = new THREE.Color(0x1e90ff);
				bottomColor = new THREE.Color(0xb0e0e6);
				sunLight.intensity = 2.5;
			}
			
			skyMesh.material.uniforms.topColor.value = topColor;
			skyMesh.material.uniforms.bottomColor.value = bottomColor;
			skyMesh.material.uniforms.sunIntensity.value = sunIntensity;
		}
	}

	function updateTimeDisplay() {
		const now = new Date();
		const solarTime = calculateSolarTime();
		
		const hours = String(now.getHours()).padStart(2, '0');
		const minutes = String(now.getMinutes()).padStart(2, '0');
		const seconds = String(now.getSeconds()).padStart(2, '0');
		currentTimeText = `当前时间: ${hours}:${minutes}:${seconds}`;
		
		const solarHours = Math.floor(solarTime);
		const solarMinutes = Math.floor((solarTime - solarHours) * 60);
		const solarSeconds = Math.floor(((solarTime - solarHours) * 60 - solarMinutes) * 60);
		currentSolarTimeText = `真太阳时: ${String(solarHours).padStart(2, '0')}:${String(solarMinutes).padStart(2, '0')}:${String(solarSeconds).padStart(2, '0')}`;
	}

	function animate() {
		if (!browser || !renderer || !scene || !camera || !controls) return;

		animationId = requestAnimationFrame(animate);
		
		updateSunAndSky();
		updateTimeDisplay();
		controls.update();
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
	<div class="time-panel">
		<div class="time-display">{currentTimeText}</div>
		<div class="solar-time-display">{currentSolarTimeText}</div>
		<div class="location-info">位置: 北京 (纬度: {latitude}°, 经度: {longitude}°)</div>
	</div>
	
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
	}

	.sundial-container :global(canvas) {
		display: block;
		width: 100%;
		height: 100%;
	}

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
