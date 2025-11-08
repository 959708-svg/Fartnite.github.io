<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>First Person Building</title>
    <style>
        body {
            margin: 0;
            background-color: #1a1a1a;
            overflow: hidden; /* Hide scrollbars */
            font-family: Arial, sans-serif;
        }
        canvas {
            display: block;
        }
        #blocker {
            position: absolute;
            width: 100%;
            height: 100%;
            top: 0;
            left: 0;
            background-color: rgba(0, 0, 0, 0.7);
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
            flex-direction: column;
        }
        #blocker h1 {
            margin: 0 0 20px;
            font-size: 32px;
        }
        #blocker p {
            font-size: 16px;
            color: #ccc;
            line-height: 1.5;
            text-align: left; /* Align controls text */
        }
    </style>

    <!-- 0. Import Map - Updated library versions -->
    <script type="importmap">
    {
        "imports": {
            "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
            "PointerLockControls": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/controls/PointerLockControls.js",
            "Reflector": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/objects/Reflector.js",
            "cannon-es": "https://cdn.jsdelivr.net/npm/cannon-es@0.20.0/dist/cannon-es.js"
        }
    }
    </script>
</head>
<body>
    <div id="blocker">
        <div>
            <h1>Click to Start</h1>
            <p>
                <b>WASD:</b> Move<br>
                <b>Mouse:</b> Look<br>
                <b>Right-Click:</b> Use Tool
                <br><br>
                <b>1:</b> White Block (Solid)<br>
                <b>2:</b> Water (Spills Balls)<br>
                <b>3:</b> Bomb (Shatters Blocks)<br>
                <b>4:</b> Lava (Melts Blocks)<br>
                <b>5:</b> Acid (Dissolves Blocks)<br>
                <b>6:</b> Flame (Ignites Blocks)<br>
                <b>7:</b> Lightning (Strike)<br>
                <b>8:</b> Gasoline (Spills Liquid)<br>
                <b>9:</b> Wood (Burns)<br>
                <b>0:</b> Stone (Strong)
            </p>
        </div>
    </div>

    <!-- Main Application Logic (Module) -->
    <script type="module">
        import * as THREE from 'three';
        import { PointerLockControls } from 'PointerLockControls';
        import { Reflector } from 'Reflector'; 
        import * as CANNON from 'cannon-es'; // Import physics engine

        // --- Particle Shader Code ---
        const particleVertexShader = `
            uniform float uTime;
            attribute vec3 aVelocity;
            attribute float aStartTime;
            varying float vAge;
            varying float vOpacity;

            void main() {
                float timeElapsed = uTime - aStartTime;
                vAge = timeElapsed / 1.5; // Particle lives for 1.5 seconds

                // Simple physics: p = p0 + v0*t + 0.5*g*t^2
                vec3 gravity = vec3(0.0, -2.0, 0.0); // Less gravity
                vec3 newPosition = position + aVelocity * timeElapsed + 0.5 * gravity * timeElapsed * timeElapsed;

                gl_Position = projectionMatrix * modelViewMatrix * vec4(newPosition, 1.0);
                
                float size = 15.0 * (1.0 - vAge); // Shrink as it ages
                gl_PointSize = max(0.0, size); // Clamp size
                vOpacity = 1.0 - vAge;
            }
        `;

        const particleFragmentShader = `
            uniform vec3 uColor;
            varying float vOpacity;

            void main() {
                if (vOpacity < 0.0) discard; // Die after 1.5 seconds
                float alpha = vOpacity * 0.9; // Fade out
                
                // Circular particle
                float dist = length(gl_PointCoord - vec2(0.5));
                if (dist > 0.5) discard;

                gl_FragColor = vec4(uColor, alpha);
            }
        `;

        // --- Global Variables ---
        let scene, camera, renderer, controls, clock, groundMirror;
        const keyboard = {};
        const playerSpeed = 5.0;
        const playerHeight = 1.7; // Approx eye height

        // --- Physics Variables ---
        let world;
        let groundMaterial, ballMaterial, whiteBlockMaterialPhysics;
        let lavaBallMaterialPhysics, acidBallMaterialPhysics, flameBallMaterialPhysics; // Added acid/flame
        let gasolineBallMaterialPhysics; // New
        const solidBlocks = []; // Stores {mesh, body} for solid blocks
        const allBalls = [];    // Stores {mesh, body} for water balls
        const allDebris = [];   // Stores {mesh,body} for shattered pieces
        const allParticles = []; // Stores {particles} for effects
        const activeFires = []; // Stores {origin, interval, timeout} for gasoline fires
        let ballGeometry, ballMeshMaterial, ballShape;
        let lavaBallMaterial, acidBallMaterial, flameBallMaterial; // Added acid/flame
        let gasolineBallMaterial; // New
        let woodMaterial, stoneMaterial; // New building materials
 
        // --- Debris Variables ---
        const debrisGeometries = [];
        const debrisShapes = [];
        let debrisMaterial, debrisPhysicsMaterial;
        
        // --- Block Building Variables ---
        let blockGeometry, whiteBlockMaterial;
        let meltingBlockMaterial, dissolvingBlockMaterial, burningBlockMaterial; // Added new block states
        let currentBlockType = 'white'; // Start with white block
        const raycaster = new THREE.Raycaster();
        const pointer = new THREE.Vector2(0, 0); // Center of the screen


        // --- Core Functions ---

        /**
         * Initializes the CANNON.js physics world.
         */
        function initPhysics() {
            world = new CANNON.World();
            world.gravity.set(0, -9.82, 0); // Standard gravity

            // Materials
            groundMaterial = new CANNON.Material("groundMaterial");
            ballMaterial = new CANNON.Material("ballMaterial"); // Water
            lavaBallMaterialPhysics = new CANNON.Material("lavaBallMaterial");
            acidBallMaterialPhysics = new CANNON.Material("acidBallMaterial"); // New
            flameBallMaterialPhysics = new CANNON.Material("flameBallMaterial"); // New
            whiteBlockMaterialPhysics = new CANNON.Material("whiteBlockMaterial");
            debrisPhysicsMaterial = new CANNON.Material("debrisPhysicsMaterial");
            gasolineBallMaterialPhysics = new CANNON.Material("gasolineBallMaterial"); // New

            // Contact Materials (how things bounce)
            const ballGroundContact = new CANNON.ContactMaterial(ballMaterial, groundMaterial, {
                friction: 0.3,
                restitution: 0.4 // A bit bouncy
            });
            world.addContactMaterial(ballGroundContact);

            const ballBlockContact = new CANNON.ContactMaterial(ballMaterial, whiteBlockMaterialPhysics, {
                friction: 0.3,
                restitution: 0.4
            });
            world.addContactMaterial(ballBlockContact);

            // Lava Ball + Ground
            const lavaGroundContact = new CANNON.ContactMaterial(lavaBallMaterialPhysics, groundMaterial, {
                friction: 0.1,
                restitution: 0.2 // Not very bouncy
            });
            world.addContactMaterial(lavaGroundContact);

            // Lava Ball + White Block
            const lavaBlockContact = new CANNON.ContactMaterial(lavaBallMaterialPhysics, whiteBlockMaterialPhysics, {
                friction: 0,
                restitution: 0.1
            });
            world.addContactMaterial(lavaBlockContact);

            // Acid Ball + Ground
            const acidGroundContact = new CANNON.ContactMaterial(acidBallMaterialPhysics, groundMaterial, {
                friction: 0.5,
                restitution: 0.1 // Not bouncy
            });
            world.addContactMaterial(acidGroundContact);

            // Acid Ball + White Block
            const acidBlockContact = new CANNON.ContactMaterial(acidBallMaterialPhysics, whiteBlockMaterialPhysics, {
                friction: 0.2,
                restitution: 0.1
            });
            world.addContactMaterial(acidBlockContact);

            // Flame Ball + Ground
            const flameGroundContact = new CANNON.ContactMaterial(flameBallMaterialPhysics, groundMaterial, {
                friction: 0.1,
                restitution: 0.6 // Bouncy
            });
            world.addContactMaterial(flameGroundContact);

            // Flame Ball + White Block
            const flameBlockContact = new CANNON.ContactMaterial(flameBallMaterialPhysics, whiteBlockMaterialPhysics, {
                friction: 0.1,
                restitution: 0.5
            });
            world.addContactMaterial(flameBlockContact);


            // Gasoline Ball Contacts (new)
            const gasGroundContact = new CANNON.ContactMaterial(gasolineBallMaterialPhysics, groundMaterial, {
                friction: 0.3,
                restitution: 0.4 // Like water
            });
            world.addContactMaterial(gasGroundContact);

            const gasBlockContact = new CANNON.ContactMaterial(gasolineBallMaterialPhysics, whiteBlockMaterialPhysics, {
                friction: 0.3,
                restitution: 0.4
            });
            world.addContactMaterial(gasBlockContact);


            // Debris Contacts
            const debrisGroundContact = new CANNON.ContactMaterial(debrisPhysicsMaterial, groundMaterial, {
                friction: 0.4,
                restitution: 0.1 // Not very bouncy
            });
            world.addContactMaterial(debrisGroundContact);

            const debrisBallContact = new CANNON.ContactMaterial(debrisPhysicsMaterial, ballMaterial, {
                friction: 0.1,
                restitution: 0.3
            });
            world.addContactMaterial(debrisBallContact);

            const debrisDebrisContact = new CANNON.ContactMaterial(debrisPhysicsMaterial, debrisPhysicsMaterial, {
                friction: 0.5,
                restitution: 0.01
            });
            world.addContactMaterial(debrisDebrisContact);
        }

        function init() {
            // Clock
            clock = new THREE.Clock();

            // Setup Physics
            initPhysics();

            // Scene
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x000000); // Changed to black
            scene.fog = new THREE.Fog(0x000000, 1, 50); // Changed to black

            // Camera
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, playerHeight, 5); // Start at player height, back a bit
            
            // Renderer
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(window.devicePixelRatio);
            document.body.appendChild(renderer.domElement);

            // Lighting
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
            scene.add(ambientLight);
            
            const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
            dirLight.position.set(5, 10, 5);
            dirLight.castShadow = true; // Enable shadows for balls
            scene.add(dirLight);

            // --- Controls (Pointer Lock) ---
            controls = new PointerLockControls(camera, document.body);
            scene.add(controls.getObject());

            const blocker = document.getElementById('blocker');
            blocker.addEventListener('click', () => {
                // We wrap this in a try...catch as a safeguard against API errors
                try {
                    controls.lock();
                } catch (e) {
                    console.error("PointerLockControls error:", e);
                }
            });
            controls.addEventListener('lock', () => {
                blocker.style.display = 'none';
            });
            controls.addEventListener('unlock', () => {
                blocker.style.display = 'flex';
            });


            // --- Create Central Land Platform (Visual + Physics) ---
            const landSize = 20; // 20x20 platform
            const landGeometry = new THREE.PlaneGeometry(landSize, landSize);
            
            // 1. The reflective surface (Visual)
            groundMirror = new Reflector(landGeometry, {
                clipBias: 0.003,
                textureWidth: window.innerWidth * window.devicePixelRatio,
                textureHeight: window.innerHeight * window.devicePixelRatio,
                color: 0x888888, // The weak reflection color
                recursion: 1 
            });
            groundMirror.rotation.x = -Math.PI / 2;
            groundMirror.position.y = 0.05; // Slightly above 0
            scene.add(groundMirror);
            
            // 2. The grid lines (Visual)
            const gridHelper = new THREE.GridHelper(landSize, landSize, 0xffffff, 0xffffff);
            gridHelper.material.opacity = 0.25;
            gridHelper.material.transparent = true;
            gridHelper.position.y = 0.06; // Just on top of the reflector
            scene.add(gridHelper);

            // 3. The ground (Physics)
            const groundShape = new CANNON.Plane();
            const groundBody = new CANNON.Body({ 
                mass: 0, // Static
                shape: groundShape, 
                material: groundMaterial 
            });
            groundBody.quaternion.setFromEuler(-Math.PI / 2, 0, 0); // Match visual plane
            groundBody.position.set(0, 0.05, 0); // Match visual plane
            world.addBody(groundBody);


            // --- Initialize Block Materials (for placing) ---
            blockGeometry = new THREE.BoxGeometry(1, 1, 1);
            whiteBlockMaterial = new THREE.MeshLambertMaterial({ color: 0xffffff });
            
            // --- Initialize Ball Materials (for spilling) ---
            // Water
            ballGeometry = new THREE.SphereGeometry(0.1, 16, 12); // Small ball
            ballMeshMaterial = new THREE.MeshLambertMaterial({ 
                color: 0x88DDFF, // Light blue
                transparent: true,
                opacity: 0.6 
            });
            ballShape = new CANNON.Sphere(0.1); // Radius must match geometry

            // Lava
            lavaBallMaterial = new THREE.MeshLambertMaterial({
                color: 0xff6600, // Bright orange
                emissive: 0xff4400, // Makes it glow
                transparent: true,
                opacity: 0.8
            });
            meltingBlockMaterial = new THREE.MeshLambertMaterial({
                color: 0xff8800,
                emissive: 0xff6600, // Make the melting block glow
            });

            // Acid
            acidBallMaterial = new THREE.MeshLambertMaterial({
                color: 0x00ff00, // Bright green
                emissive: 0x00cc00, // Makes it glow
                transparent: true,
                opacity: 0.7
            });
            dissolvingBlockMaterial = new THREE.MeshLambertMaterial({
                color: 0x00ff00,
                emissive: 0x00ff00, // Make the dissolving block glow
            });
            
            // Gasoline (New)
            gasolineBallMaterial = new THREE.MeshLambertMaterial({ 
                color: 0xcccc99, // Yellowish-brown
                transparent: true,
                opacity: 0.5 
            });

            // Flame
            flameBallMaterial = new THREE.MeshLambertMaterial({
                color: 0xff3300, // Bright red
                emissive: 0xee2200, // Makes it glow
            });
            burningBlockMaterial = new THREE.MeshLambertMaterial({
                color: 0x222222, // Charred black
                emissive: 0xff3300, // Glowing embers
            });

            // Wood (New)
            woodMaterial = new THREE.MeshLambertMaterial({ color: 0x8B4513 }); // Brown
            
            // Stone (New)
            stoneMaterial = new THREE.MeshLambertMaterial({ color: 0x808080 }); // Grey


            // --- Initialize Debris Materials (for shattering) ---
            debrisMaterial = new THREE.MeshLambertMaterial({ color: 0xcccccc }); // Grey/white debris
            // 1. Small Box
            debrisGeometries.push(new THREE.BoxGeometry(0.3, 0.3, 0.3));
            debrisShapes.push(new CANNON.Box(new CANNON.Vec3(0.15, 0.15, 0.15)));
            // 2. Small Sphere (Circle)
            debrisGeometries.push(new THREE.SphereGeometry(0.2, 8, 6));
            debrisShapes.push(new CANNON.Sphere(0.2));
            // 3. Small Cone (Triangle/Polygon)
            debrisGeometries.push(new THREE.ConeGeometry(0.2, 0.4, 8));
            debrisShapes.push(new CANNON.Cylinder(0.01, 0.2, 0.4, 8)); // Approx. shape

            // --- Event Listeners ---
            window.addEventListener('resize', onWindowResize);
            
            document.addEventListener('keydown', (event) => {
                keyboard[event.code] = true;
                
                // --- Block Switching Logic ---
                if (event.code === 'Digit1') currentBlockType = 'white';
                if (event.code === 'Digit2') currentBlockType = 'water';
                if (event.code === 'Digit3') currentBlockType = 'bomb';
                if (event.code === 'Digit4') currentBlockType = 'lava';
                if (event.code === 'Digit5') currentBlockType = 'acid'; // Added
                if (event.code === 'Digit6') currentBlockType = 'flame'; // Added
                if (event.code === 'Digit7') currentBlockType = 'lightning'; // Added
                if (event.code === 'Digit8') currentBlockType = 'gasoline'; // New
                if (event.code === 'Digit9') currentBlockType = 'wood'; // New
                if (event.code === 'Digit0') currentBlockType = 'stone'; // New
            });
            
            document.addEventListener('keyup', (event) => (keyboard[event.code] = false));
            
            document.addEventListener('mousedown', (event) => {
                if (controls.isLocked && event.button === 2) { // Right-click
                    placeBlockOrWater();
                }
            });

            document.addEventListener('contextmenu', (event) => event.preventDefault());
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        /**
         * Generic function to spill a group of balls.
         */
        function spillGenericBallGroup(origin, type) {
            const ballCount = 100;
            const ballLifetime = 10000; // 10 seconds

            let visualMaterial, physicsMaterial, tag = null;

            switch (type) {
                case 'water':
                    visualMaterial = ballMeshMaterial;
                    physicsMaterial = ballMaterial;
                    tag = 'isWater'; // Add tag to identify water
                    break;
                case 'lava':
                    visualMaterial = lavaBallMaterial;
                    physicsMaterial = lavaBallMaterialPhysics;
                    tag = 'isLava';
                    break;
                case 'acid':
                    visualMaterial = acidBallMaterial;
                    physicsMaterial = acidBallMaterialPhysics;
                    tag = 'isAcid';
                    break;
                case 'gasoline': // New
                    visualMaterial = gasolineBallMaterial;
                    physicsMaterial = gasolineBallMaterialPhysics;
                    tag = 'isGasoline';
                    break;
                case 'flame':
                    visualMaterial = flameBallMaterial;
                    physicsMaterial = flameBallMaterialPhysics;
                    tag = 'isFlame';
                    break;
                default:
                    return; // Unknown type
            }


            for (let i = 0; i < ballCount; i++) {
                // Create visual mesh
                const ballMesh = new THREE.Mesh(ballGeometry, visualMaterial);
                ballMesh.castShadow = true;
                scene.add(ballMesh);

                // Create physics body
                const ballBody = new CANNON.Body({
                    mass: 0.1, 
                    shape: ballShape,
                    material: physicsMaterial
                });
                
                // Set initial position with a "group" cluster
                const clusterOffset = new THREE.Vector3(
                    (Math.random() - 0.5) * 0.5,
                    (Math.random() - 0.5) * 0.5,
                    (Math.random() - 0.5) * 0.5
                );
                
                ballBody.position.set(
                    origin.x + clusterOffset.x,
                    origin.y + clusterOffset.y,
                    origin.z + clusterOffset.z
                );
                
                if (tag) ballBody[tag] = true; // Tag this body (e.g., body.isLava = true)
                world.addBody(ballBody);
                
                // Store for update loop
                const ball = { mesh: ballMesh, body: ballBody };
                allBalls.push(ball); // Add to the same array for simplicity

                // Optimization: Remove ball after lifetime
                setTimeout(() => {
                    // Check if body and mesh still exist before removing
                    if (world.bodies.includes(ballBody)) {
                        world.removeBody(ballBody);
                    }
                    if (scene.children.includes(ballMesh)) {
                        scene.remove(ballMesh);
                    }
                    ballMesh.geometry.dispose(); // Clean up geometry
                    
                    const index = allBalls.indexOf(ball);
                    if (index > -1) allBalls.splice(index, 1);
                }, ballLifetime + Math.random() * 2000); // Stagger removal
            }
        }
        
        // --- Remove old spill functions ---
        // We no longer need spillBallGroup or spillLavaGroup
        // They are replaced by spillGenericBallGroup


        /**
         * Turns a block orange and makes it disappear after a delay. (Used by Lava)
         */
        function meltBlock(blockToMelt) {
            // 1. Flag it so we don't melt it twice
            blockToMelt.isMelting = true;
            
            // 2. Clear other reactions
            if (blockToMelt.burnInterval) putOutFire(blockToMelt); // Put out fire if it's burning
            if (blockToMelt.dissolveTimeout) clearTimeout(blockToMelt.dissolveTimeout);
            
            // 3. Change its material to glowing orange
            blockToMelt.mesh.material = meltingBlockMaterial;

            // 4. Set a timeout to remove it
            blockToMelt.meltTimeout = setTimeout(() => {
                // Check if it hasn't been shattered/dissolved in the meantime
                if (world.bodies.includes(blockToMelt.body)) {
                    world.removeBody(blockToMelt.body);
                }
                if (scene.children.includes(blockToMelt.mesh)) {
                    scene.remove(blockToMelt.mesh);
                }
                blockToMelt.mesh.geometry.dispose();
                
                const index = solidBlocks.indexOf(blockToMelt);
                if (index > -1) solidBlocks.splice(index, 1);
                
            }, 2000); // 2-second melt time
        }
        
        /**
         * Turns a block green and makes it disappear after a delay. (Used by Acid)
         */
        function dissolveBlock(blockToDissolve) {
            blockToDissolve.isDissolving = true;
            
            // Clear other reactions
            if (blockToDissolve.burnInterval) putOutFire(blockToDissolve); // Put out fire if it's burning
            if (blockToDissolve.meltTimeout) clearTimeout(blockToDissolve.meltTimeout);

            blockToDissolve.mesh.material = dissolvingBlockMaterial;

            blockToDissolve.dissolveTimeout = setTimeout(() => {
                if (world.bodies.includes(blockToDissolve.body)) {
                    world.removeBody(blockToDissolve.body);
                }
                if (scene.children.includes(blockToDissolve.mesh)) {
                    scene.remove(blockToDissolve.mesh);
                }
                blockToDissolve.mesh.geometry.dispose();
                
                const index = solidBlocks.indexOf(blockToDissolve);
                if (index > -1) solidBlocks.splice(index, 1);
                
            }, 2000); // 2-second dissolve time
        }

        /**
         * Turns a block black and shatters it after a delay. (Used by Flame)
         */
        function igniteBlock(blockToIgnite) {
            blockToIgnite.isBurning = true;

            // Clear other reactions
            if (blockToIgnite.meltTimeout) clearTimeout(blockToIgnite.meltTimeout);
            if (blockToIgnite.dissolveTimeout) clearTimeout(blockToIgnite.dissolveTimeout);

            blockToIgnite.mesh.material = burningBlockMaterial;
            
            const blockPosition = blockToIgnite.body.position;
            
            // Store the interval on the block object
            blockToIgnite.burnInterval = setInterval(() => {
                const randomOffset = new THREE.Vector3((Math.random() - 0.5) * 0.8, (Math.random() - 0.5) * 0.8, (Math.random() - 0.5) * 0.8);
                const fireOrigin = new THREE.Vector3().copy(blockPosition).add(randomOffset);
                createExplosionParticles(fireOrigin, 0xff6600); // Orange flame
            }, 500); // New particle poof every 0.5s

            // Store the timeout on the block object
            blockToIgnite.burnTimeout = setTimeout(() => {
                clearInterval(blockToIgnite.burnInterval); // Stop fire particles
                // Check if it still exists before shattering
                if (world.bodies.includes(blockToIgnite.body)) {
                    shatterBlock(blockToIgnite, blockToIgnite.body.position);
                    
                    const index = solidBlocks.indexOf(blockToIgnite);
                    if (index > -1) solidBlocks.splice(index, 1);
                }
            }, 20000); // 20-second burn time (was 2000)
        }

        /**
         * Puts out a fire on a block
         */
        function putOutFire(block) {
            if (!block.isBurning) return;
            
            block.isBurning = false;
            clearInterval(block.burnInterval); // Clear the fire particle interval
            block.burnInterval = null;
            
            // Clear the 20-second shatter timeout
            clearTimeout(block.burnTimeout);
            block.burnTimeout = null;
            
            // Revert material from "burning" to "original"
            block.mesh.material = block.originalMaterial; 
            
            // Create a "steam" particle effect
            createExplosionParticles(block.body.position, 0xcccccc); // White/grey steam
        }


        /**
         * Shatters a block into 70-130 random pieces.
         */
        function shatterBlock(block, explosionOrigin) {
            // Remove the original solid block
            world.removeBody(block.body);
            scene.remove(block.mesh);
            block.mesh.geometry.dispose();

            const numParts = Math.floor(Math.random() * 60 + 70); // 70 to 130
            const blockOrigin = block.body.position;
            const debrisLifetime = 8000; // 8 seconds

            for (let i = 0; i < numParts; i++) {
                // Pick a random debris shape
                const shapeIndex = Math.floor(Math.random() * debrisGeometries.length);
                const randomGeom = debrisGeometries[shapeIndex];
                const randomShape = debrisShapes[shapeIndex];
                
                // Create visual mesh
                const debrisMesh = new THREE.Mesh(randomGeom, debrisMaterial);
                debrisMesh.castShadow = true;
                scene.add(debrisMesh);

                // Create physics body
                const debrisBody = new CANNON.Body({
                    mass: 0.1,
                    shape: randomShape,
                    material: debrisPhysicsMaterial
                });
                
                // Start at the block's old position, slightly randomized
                debrisBody.position.set(
                    blockOrigin.x + (Math.random() - 0.5) * 0.5,
                    blockOrigin.y + (Math.random() - 0.5) * 0.5,
                    blockOrigin.z + (Math.random() - 0.5) * 0.5
                );
                world.addBody(debrisBody);

                // Apply explosion force
                const forceDirection = new CANNON.Vec3();
                debrisBody.position.vsub(explosionOrigin, forceDirection); // Vector from explosion to debris
                forceDirection.normalize();
                const forceMagnitude = Math.random() * 4 + 2; // Random force
                forceDirection.scale(forceMagnitude, forceDirection);
                debrisBody.applyImpulse(forceDirection, debrisBody.position); // Apply force

                // Store for update loop
                const debrisPiece = { mesh: debrisMesh, body: debrisBody };
                allDebris.push(debrisPiece);

                // Optimization: Remove debris after lifetime
                setTimeout(() => {
                    if (world.bodies.includes(debrisBody)) world.removeBody(debrisBody);
                    if (scene.children.includes(debrisMesh)) scene.remove(debrisMesh);
                    // Note: Don't dispose geometry since it's shared
                    
                    const index = allDebris.indexOf(debrisPiece);
                    if (index > -1) allDebris.splice(index, 1);
                }, debrisLifetime + Math.random() * 2000);
            }
        }

        /**
         * Applies an explosion force to nearby water balls.
         * Now accepts optional strength and radius.
         */
        function applyExplosionForce(origin, strength = 6, radius = 5) {
            const explosionRadius = radius; // Affects balls within 5 meters
            const explosionStrength = strength;

            for (const ball of allBalls) {
                const dist = ball.body.position.distanceTo(origin);

                if (dist < explosionRadius) {
                    const forceDirection = new CANNON.Vec3();
                    ball.body.position.vsub(origin, forceDirection); // Vector from explosion to ball
                    forceDirection.normalize();
                    
                    // Force is weaker further away
                    const forceMagnitude = (1 - (dist / explosionRadius)) * explosionStrength;
                    forceDirection.scale(forceMagnitude, forceDirection);

                    ball.body.applyImpulse(forceDirection, ball.body.position);
                }
            }
        }

        /**
         * Checks for and ignites nearby gasoline balls.
         */
        function igniteGasoline(origin) {
            const ignitionRadius = 4; // Check in a 4m radius
            let gasolineIgnited = false;
            const ballsToRemove = [];

            for (const ball of allBalls) {
                // Check if it's gasoline and hasn't been checked this frame
                if (ball.body.isGasoline) {
                    const dist = ball.body.position.distanceTo(origin);
                    if (dist < ignitionRadius) {
                        gasolineIgnited = true;
                        ballsToRemove.push(ball); // Mark for removal
                    }
                }
            }
            
            if (gasolineIgnited) {
                // "tons of fire and a big one"
                // 1. Create a BIG particle effect
                createExplosionParticles(origin, 0xff3300); // Big fire
                createExplosionParticles(origin, 0xffaa33); // Sparks
                createExplosionParticles(origin, 0x555555); // Smoke
                
                // 2. Create a BIG explosion force (Strength 15, Radius 8)
                applyExplosionForce(origin, 15, 8); 
                
                // 3. Remove the gasoline balls that just ignited
                for (const ball of ballsToRemove) {
                    if (world.bodies.includes(ball.body)) world.removeBody(ball.body);
                    if (scene.children.includes(ball.mesh)) scene.remove(ball.mesh);
                    ball.mesh.geometry.dispose(); // Clean up
                    
                    const index = allBalls.indexOf(ball);
                    if (index > -1) allBalls.splice(index, 1);
                }
                
                // 4. Add the persistent fire
                const fireInterval = setInterval(() => {
                    createExplosionParticles(origin, 0xff6600); // Continuous fire
                }, 500); // New poof every 0.5s
                
                const fireTimeout = setTimeout(() => {
                    clearInterval(fireInterval);
                    // Remove from activeFires array
                    const fireObj = activeFires.find(f => f.interval === fireInterval);
                    if (fireObj) {
                        const index = activeFires.indexOf(fireObj);
                        if (index > -1) activeFires.splice(index, 1);
                    }
                }, 20000); // Stop after 20 seconds
                
                // Store this fire
                activeFires.push({ origin: origin, interval: fireInterval, timeout: fireTimeout });
            }
            
            return gasolineIgnited; // Return true if we started a fire
        }


        /**
         * Creates GPU-based particle effects for an explosion.
         * Accepts a color for the particles.
         */
        function createExplosionParticles(origin, color) {
            const particleCount = 200;
            const particleLifetime = 1500; // 1.5 seconds (matches shader)
            
            const positions = new Float32Array(particleCount * 3);
            const velocities = new Float32Array(particleCount * 3);
            const startTimes = new Float32Array(particleCount);
            
            const particleGeom = new THREE.BufferGeometry();
            const time = clock.getElapsedTime();

            for (let i = 0; i < particleCount; i++) {
                // Start all particles at the origin
                positions[i * 3 + 0] = origin.x;
                positions[i * 3 + 1] = origin.y;
                positions[i * 3 + 2] = origin.z;

                // Random velocity for "spark" / "smoke"
                const vec = new THREE.Vector3(
                    (Math.random() - 0.5),
                    (Math.random() - 0.5) + 0.5, // Biased upwards
                    (Math.random() - 0.5)
                );
                vec.normalize().multiplyScalar(Math.random() * 5 + 1); // Speed
                
                velocities[i * 3 + 0] = vec.x;
                velocities[i * 3 + 1] = vec.y;
                velocities[i * 3 + 2] = vec.z;
                
                startTimes[i] = time + Math.random() * 0.1; // Stagger start
            }

            particleGeom.setAttribute('position', new THREE.BufferAttribute(positions, 3));
            particleGeom.setAttribute('aVelocity', new THREE.BufferAttribute(velocities, 3));
            particleGeom.setAttribute('aStartTime', new THREE.BufferAttribute(startTimes, 1));
            
            // Material (re-used shader, new uniforms)
            const particleMaterial = new THREE.ShaderMaterial({
                uniforms: {
                    uTime: { value: 0.0 },
                    uColor: { value: new THREE.Color(color) }, // Use passed color
                },
                vertexShader: particleVertexShader,
                fragmentShader: particleFragmentShader,
                transparent: true,
                blending: THREE.AdditiveBlending,
                depthWrite: false
            });
            
            const particles = new THREE.Points(particleGeom, particleMaterial);
            scene.add(particles);
            allParticles.push(particles);

            // Optimization: Remove particles after lifetime
            setTimeout(() => {
                scene.remove(particles);
                particleGeom.dispose();
                particleMaterial.dispose();
                
                const index = allParticles.indexOf(particles);
                if (index > -1) allParticles.splice(index, 1);
            }, particleLifetime);
        }


        function placeBlockOrWater() {
            // Raycast from the center of the camera
            raycaster.setFromCamera(pointer, camera);

            // Check for intersections with the platform and existing blocks
            const objectsToIntersect = [groundMirror, ...solidBlocks.map(b => b.mesh)];
            const intersects = raycaster.intersectObjects(objectsToIntersect);

            if (intersects.length > 0) {
                const intersect = intersects[0];
                const clickPoint = intersect.point; // Use this as the main intersection point
                
                // --- Check if we clicked a solid block ---
                const clickedBlock = solidBlocks.find(b => b.mesh === intersect.object);

                // --- Calculate spill position (for liquids) ---
                const spillPosition = new THREE.Vector3().copy(clickPoint);
                spillPosition.addScaledVector(intersect.face.normal, 1.0); // Start 1m above
                
                if (currentBlockType === 'white' || currentBlockType === 'wood' || currentBlockType === 'stone') {
                    // --- Place a solid block (White, Wood, or Stone) ---
                    
                    // Calculate the new block's center position
                    const newBlockCenter = new THREE.Vector3().copy(intersect.point);
                    newBlockCenter.addScaledVector(intersect.face.normal, 0.5);

                    // Snap this center to a 1x1 grid
                    newBlockCenter.x = Math.floor(newBlockCenter.x) + 0.5;
                    newBlockCenter.y = Math.floor(newBlockCenter.y) + 0.5;
                    newBlockCenter.z = Math.floor(newBlockCenter.z) + 0.5;
                    
                    if (newBlockCenter.y < 0.5) newBlockCenter.y = 0.5;

                    // --- Select appropriate material ---
                    let visualMaterial;
                    let blockTypeTag = 'white'; // default
                    
                    if (currentBlockType === 'wood') {
                        visualMaterial = woodMaterial;
                        blockTypeTag = 'wood';
                    } else if (currentBlockType === 'stone') {
                        visualMaterial = stoneMaterial;
                        blockTypeTag = 'stone';
                    } else {
                        visualMaterial = whiteBlockMaterial;
                    }
                    
                    // Create visual mesh
                    const blockMesh = new THREE.Mesh(blockGeometry, visualMaterial);
                    blockMesh.position.copy(newBlockCenter);
                    scene.add(blockMesh);
                    
                    // Add physics body for the white block
                    const blockShape = new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5));
                    const blockBody = new CANNON.Body({
                        mass: 0, // Static, doesn't move
                        shape: blockShape,
                        material: whiteBlockMaterialPhysics
                    });
                    blockBody.position.set(newBlockCenter.x, newBlockCenter.y, newBlockCenter.z);
                    world.addBody(blockBody);
                    
                    // --- Link our game object to the physics body ---
                    const block = { 
                        mesh: blockMesh, 
                        body: blockBody, 
                        blockType: blockTypeTag, // Tag 'white', 'wood', or 'stone'
                        originalMaterial: visualMaterial, // Store this for putting out fire
                        isMelting: false,
                        isDissolving: false, // Add new flags
                        isBurning: false,
                        burnInterval: null,   // For persistent fire particles
                        burnTimeout: null,    // For shatter
                        meltTimeout: null,    // For melting
                        dissolveTimeout: null // For dissolving
                    };
                    blockBody.isBlock = true; // Tag the physics body
                    blockBody.gameBlock = block; // Add a reference back to our object
                    
                    // Add collision listener to this specific block
                    blockBody.addEventListener("collide", (event) => {
                        const collidedBody = event.body;
                        
                        // Check for Lava (Melts White and Wood, but not Stone)
                        if (collidedBody && collidedBody.isLava && 
                            block.blockType !== 'stone' && 
                            !block.isMelting && !block.isDissolving && !block.isBurning) {
                            meltBlock(block); // Pass the game block object
                        }
                        
                        // Check for Acid (Dissolves everything)
                        if (collidedBody && collidedBody.isAcid && !block.isMelting && !block.isDissolving && !block.isBurning) {
                            dissolveBlock(block);
                        }

                        // Check for Flame (Ignites White and Wood, but not Stone)
                        if (collidedBody && collidedBody.isFlame && 
                            block.blockType !== 'stone' && 
                            !block.isMelting && !block.isDissolving && !block.isBurning) {
                            igniteBlock(block);
                        }
                        
                        // Check for Water (Puts out fire)
                        if (collidedBody && collidedBody.isWater && block.isBurning) {
                            putOutFire(block);
                        }
                    });
                    // ---
                    
                    solidBlocks.push(block); // Push the full object

                } else if (currentBlockType === 'water') {
                    // --- Spill a group of water balls ---
                    
                    // --- CHECK FOR FIRES TO PUT OUT ---
                    const waterRadius = 3; // 3m radius to put out fires
                    
                    // 1. Check burning blocks
                    for (const block of solidBlocks) {
                        if (block.isBurning) {
                            const dist = block.body.position.distanceTo(clickPoint);
                            if (dist < waterRadius) {
                                putOutFire(block);
                            }
                        }
                    }
                    
                    // 2. Check burning gasoline puddles
                    const firesToPutOut = [];
                    for (const fire of activeFires) {
                        const dist = fire.origin.distanceTo(clickPoint);
                        if (dist < waterRadius) {
                            firesToPutOut.push(fire);
                        }
                    }
                    
                    for (const fire of firesToPutOut) {
                        clearInterval(fire.interval);
                        clearTimeout(fire.timeout);
                        const index = activeFires.indexOf(fire);
                        if (index > -1) activeFires.splice(index, 1);
                        // Create steam
                        createExplosionParticles(fire.origin, 0xcccccc);
                    }
                    // ---
                    
                    spillGenericBallGroup(spillPosition, 'water');
                
                } else if (currentBlockType === 'lava') {
                    // --- Spill a group of lava balls ---
                    spillGenericBallGroup(spillPosition, 'lava');
                
                } else if (currentBlockType === 'acid') {
                    // --- Spill a group of acid balls ---
                    spillGenericBallGroup(spillPosition, 'acid');
                
                } else if (currentBlockType === 'gasoline') {
                    // --- Spill a group of gasoline balls ---
                    spillGenericBallGroup(spillPosition, 'gasoline');

                } else if (currentBlockType === 'flame') {
                    // --- Use the Flame Tool (Not a liquid) ---
                    
                    // 1. Create a "poof" of fire particles at the click point
                    createExplosionParticles(clickPoint, 0xff6600); // Fiery orange
                    
                    // 2. Check if we ignited any gasoline
                    const fireStarted = igniteGasoline(clickPoint);

                    // 3. If no fire, try to ignite a block (if it's not stone)
                    if (!fireStarted && clickedBlock && 
                        clickedBlock.blockType !== 'stone' &&
                        !clickedBlock.isMelting && !clickedBlock.isDissolving && !clickedBlock.isBurning) {
                        igniteBlock(clickedBlock);
                    }
                
                } else if (currentBlockType === 'bomb') {
                    // --- Use the Bomb Tool ---
                    const explosionOrigin = clickPoint;

                    // Create visual particle effects
                    createExplosionParticles(explosionOrigin, 0xffaa33); // Orange sparks
                    createExplosionParticles(explosionOrigin, 0xaaaaaa); // Grey dust

                    // Check if we clicked a solid block (using the variable from the top)
                    if (clickedBlock) {
                        // Shatter the block
                        shatterBlock(clickedBlock, explosionOrigin);
                        // Remove from the solidBlocks list
                        const index = solidBlocks.indexOf(clickedBlock);
                        if (index > -1) solidBlocks.splice(index, 1);
                    } else {
                        // Just apply force to nearby balls
                        applyExplosionForce(explosionOrigin);
                    }
                } else if (currentBlockType === 'lightning') {
                    // --- Use the Lightning Tool ---
                    const strikePoint = intersect.point;
                    
                    // 1. Create particles (CHANGED to yellow)
                    createExplosionParticles(strikePoint, 0xffff00); // Yellow spark
                    
                    // 2. Check if we ignited any gasoline
                    const fireStarted = igniteGasoline(strikePoint);

                    // 3. Apply force (shockwave)
                    applyExplosionForce(strikePoint);
                    
                    // 4. Create visual bolt (CHANGED to yellow)
                    const boltMaterial = new THREE.LineBasicMaterial({ color: 0xffff00, linewidth: 3 });
                    const boltPoints = [
                        strikePoint.clone(),
                        strikePoint.clone().add(new THREE.Vector3(0, 20, 0)) // Up 20 units
                    ];
                    const boltGeom = new THREE.BufferGeometry().setFromPoints(boltPoints);
                    const boltLine = new THREE.Line(boltGeom, boltMaterial);
                    scene.add(boltLine);
                    
                    // Remove the bolt after a short time
                    setTimeout(() => {
                        scene.remove(boltLine);
                        boltGeom.dispose();
                        boltMaterial.dispose();
                    }, 150); // 150ms bolt
                }
            }
        }

        function handleMovement(deltaTime) {
            const moveDistance = playerSpeed * deltaTime; // How far to move this frame

            if (keyboard['KeyW']) controls.moveForward(moveDistance);
            if (keyboard['KeyS']) controls.moveForward(-moveDistance); // <-- Fixed typo here
            if (keyboard['KeyA']) controls.moveRight(-moveDistance); 
            if (keyboard['KeyD']) controls.moveRight(moveDistance);
        }

        // --- Animation Loop ---
        function animate() {
            requestAnimationFrame(animate);
            
            const deltaTime = clock.getDelta();
            const elapsedTime = clock.getElapsedTime();

            // Handle movement only if controls are locked
            if (controls.isLocked === true) {
                handleMovement(deltaTime);
            }
            
            // --- Update Physics ---
            // Use a fixed timestep for stability
            world.step(1 / 60, deltaTime, 3);

            // Link ball meshes to physics bodies
            for (const ball of allBalls) {
                ball.mesh.position.set(
                    ball.body.position.x,
                    ball.body.position.y,
                    ball.body.position.z
                );
                ball.mesh.quaternion.set(
                    ball.body.quaternion.x,
                    ball.body.quaternion.y,
                    ball.body.quaternion.z,
                    ball.body.quaternion.w
                );
            }
            
            // Link debris meshes to physics bodies
            for (const debris of allDebris) {
                debris.mesh.position.set(
                    debris.body.position.x,
                    debris.body.position.y,
                    debris.body.position.z
                );
                debris.mesh.quaternion.set(
                    debris.body.quaternion.x,
                    debris.body.quaternion.y,
                    debris.body.quaternion.z,
                    debris.body.quaternion.w
                );
            }

            // Update particle shaders
            for (const particles of allParticles) {
                particles.material.uniforms.uTime.value = elapsedTime;
            }

            // Render
            renderer.render(scene, camera);
        }

        // --- Start ---
        init();
        animate();

    </script>
</body>
</html>
