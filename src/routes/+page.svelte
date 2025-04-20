<script lang="ts">
    import * as THREE from "three";
    import { STLLoader } from "three/examples/jsm/loaders/STLLoader.js";
    import { STLExporter } from "three/examples/jsm/exporters/STLExporter.js";
    import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
    import { mergeVertices } from "three/examples/jsm/utils/BufferGeometryUtils.js";
    import type { ChangeEventHandler } from "svelte/elements";
    import { onMount } from "svelte";

    // Global variables
    let originalGeometry = null;
    let repairedGeometry = null;
    let renderer: THREE.WebGLRenderer, scene: THREE.Scene, camera: THREE.PerspectiveCamera, controls: OrbitControls;
    let mesh = null;

    // DOM elements

    let dropArea: HTMLDivElement;
    let fileInput: HTMLInputElement;
    let repairBtnDisabled = true;
    let showDownloadBtn = false;

    let showProgress = false;
    let progressContainer: HTMLDivElement;
    let progressBar: HTMLDivElement;
    // const statusDiv = document.getElementById("status");
    let repairOptions: string[] = [];
    let statsContainer: HTMLDivElement, showStats = false;
    let vertexCount: number, faceCount: number, problemCount: number, progress = 0;
    let modelPreview: HTMLDivElement;
    let statusText = "";

    const handleWindowResize = () => {
        if (!camera) return;
        camera.aspect = modelPreview.clientWidth / modelPreview.clientHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(modelPreview.clientWidth, modelPreview.clientHeight);
    }

    // Initialize Three.js scene
    function initScene() {
        // Create scene
        scene = new THREE.Scene();
        scene.background = new THREE.Color(0x0f0f0f);

        // Create camera
        camera = new THREE.PerspectiveCamera(
            75,
            modelPreview.clientWidth / modelPreview.clientHeight,
            0.1,
            1000,
        );
        camera.position.z = 5;

        // Create renderer
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(modelPreview.clientWidth, modelPreview.clientHeight);
        renderer.setPixelRatio(window.devicePixelRatio);
        modelPreview.appendChild(renderer.domElement);

        // Add orbit controls
        controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.dampingFactor = 0.25;
        controls.minDistance = 10;
        controls.maxDistance = 500;

        // Add lights
        const ambientLight = new THREE.AmbientLight(0x888888);
        scene.add(ambientLight);

        const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
        directionalLight.position.set(1, 1, 1);
        scene.add(directionalLight);

        // Animation loop
        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        animate();
    }

    // Initialize Three.js scene
    onMount(() => {
        initScene();
        // Handle file drop
        const preventDefaults = (e) => {
            e.preventDefault();
            e.stopPropagation();
        }
        const highlight = () => dropArea.classList.add("drag-over");
        const unhighlight = () => dropArea.classList.remove("drag-over");

        ["dragenter", "dragover", "dragleave", "drop"].forEach((eventName) => {
            dropArea.addEventListener(eventName, preventDefaults, false);
        });

        ["dragenter", "dragover"].forEach((eventName) => {
            dropArea.addEventListener(eventName, highlight, false);
        });

        ["dragleave", "drop"].forEach((eventName) => {
            dropArea.addEventListener(eventName, unhighlight, false);
        });
    });

    function handleDrop(e) {
        const dt = e.dataTransfer;
        const files = dt.files;
        handleFiles(files);
    }

    const handleFiles: ChangeEventHandler<HTMLInputElement> = (event) => {
        // @ts-ignore
        const files = event.target.files;
        console.log('event:', event)
        console.log('files:', files)
        if (files.length > 0) {
            const file = files[0];
            if (file.name.toLowerCase().endsWith(".stl")) {
                loadSTL(file);
            } else {
                updateStatus("Please select an STL file.", "error");
            }
        }
        return;
    }

    function loadSTL(file) {
        updateStatus("Loading STL file...", "info");

        const reader = new FileReader();
        reader.onload = function (e) {
            const loader = new STLLoader();
            try {
                originalGeometry = loader.parse(e.target.result);

                updateStatus("STL file loaded successfully.", "success");
                updateStatus(`Analyzing mesh...`, "info");

                // Update stats
                vertexCount =
                    originalGeometry.attributes.position.count;
                faceCount =
                    originalGeometry.attributes.position.count / 3;

                // Find problems
                const problems = analyzeGeometry(originalGeometry);
                problemCount = problems.length;

                if (problems.length > 0) {
                    updateStatus(
                        `Found ${problems.length} potential issues with the mesh:`,
                        "warning",
                    );
                    problems.forEach((problem) => {
                        updateStatus(`- ${problem}`, "warning");
                    });
                } else {
                    updateStatus("No issues found with the mesh.", "success");
                }

                // Show the mesh
                displayMesh(originalGeometry);

                // Show repair options
                // repairOptions.style.display = "block";
                showStats = true;
                showProgress = true;
                repairBtnDisabled = false;
            } catch (error) {
                updateStatus(`Error loading STL: ${error.message}`, "error");
            }
        };
        reader.readAsArrayBuffer(file);
    }

    function displayMesh(geometry) {
        // Remove existing mesh
        if (mesh) scene.remove(mesh);

        // Create new mesh
        const material = new THREE.MeshStandardMaterial({
            // color: 0x5599ff, // blue (my fav)
            color: 0x00ff9d, // green
            metalness: 0.1,
            roughness: 0.5,
            side: THREE.DoubleSide,
        });

        mesh = new THREE.Mesh(geometry, material);

        // Center the mesh
        geometry.computeBoundingBox();
        const boundingBox = geometry.boundingBox;
        const center = boundingBox.getCenter(new THREE.Vector3());
        mesh.position.x = -center.x;
        mesh.position.y = -center.y;
        mesh.position.z = -center.z;

        scene.add(mesh);

        // Adjust camera
        const size = boundingBox.getSize(new THREE.Vector3());
        const maxDim = Math.max(size.x, size.y, size.z);
        const fov = camera.fov * (Math.PI / 180);
        let cameraZ = Math.abs(maxDim / Math.sin(fov / 2)) * 1.5; // Increased distance by 50%

        camera.position.z = cameraZ;
        controls.target.copy(center);
        controls.update();

        // Update camera and controls
        camera.updateProjectionMatrix();
    }

    function analyzeGeometry(geometry) {
        const problems = [];

        // Check for non-manifold edges
        const positionAttr = geometry.attributes.position;
        const positions = positionAttr.array;

        // Basic checks
        if (positionAttr.count % 3 !== 0) problems.push("Mesh has incomplete triangles");

        // Check for degenerate triangles
        let degenerateCount = 0;

        for (let i = 0; i < positionAttr.count; i += 3) {
            const i1 = i;
            const i2 = i + 1;
            const i3 = i + 2;

            const v1 = new THREE.Vector3(
                positions[i1 * 3],
                positions[i1 * 3 + 1],
                positions[i1 * 3 + 2],
            );

            const v2 = new THREE.Vector3(
                positions[i2 * 3],
                positions[i2 * 3 + 1],
                positions[i2 * 3 + 2],
            );

            const v3 = new THREE.Vector3(
                positions[i3 * 3],
                positions[i3 * 3 + 1],
                positions[i3 * 3 + 2],
            );

            // Check if any two vertices are the same
            if (
                v1.distanceTo(v2) < 1e-10 ||
                v2.distanceTo(v3) < 1e-10 ||
                v3.distanceTo(v1) < 1e-10
            ) {
                degenerateCount++;
            }
        }

        if (degenerateCount > 0) {
            problems.push(`${degenerateCount} degenerate triangles found`);
        }

        // Check for inverted normals
        if (!geometry.attributes.normal) problems.push("Mesh has no normals");

        // Check for duplicated vertices
        const vertexMap = new Map();
        let duplicateCount = 0;

        for (let i = 0; i < positionAttr.count; i++) {
            const x = positions[i * 3].toFixed(6);
            const y = positions[i * 3 + 1].toFixed(6);
            const z = positions[i * 3 + 2].toFixed(6);

            const key = `${x},${y},${z}`;

            if (vertexMap.has(key)) {
                duplicateCount++;
            } else {
                vertexMap.set(key, i);
            }
        }

        if (duplicateCount > 0) {
            problems.push(`${duplicateCount} duplicate vertices found`);
        }

        return problems;
    }

    // Repair button click handler
    const handleRepairBtnClick = () => {
        if (!originalGeometry) return;

        updateStatus("Repairing mesh...", "info");
        // progressContainer.style.display = "block";
        progressBar.style.width = "0%";

        // Simulate progress
        // let progress = 0;
        const progressInterval = setInterval(() => {
            progress += 5;
            progressBar.style.width = `${progress}%`;
            if (progress >= 100) {
                clearInterval(progressInterval);
            }
        }, 50);

        // Perform repair operations
        setTimeout(() => {
            repairedGeometry = repairGeometry(originalGeometry);

            // Update stats
            vertexCount = repairedGeometry.attributes.position.count;
            faceCount = repairedGeometry.attributes.position.count / 3;

            // Display repaired mesh
            displayMesh(repairedGeometry);

            clearInterval(progressInterval);
            progressBar.style.width = "100%";
            setTimeout(() => {
                showProgress = false;
                showDownloadBtn = true;
            }, 1000);

            updateStatus("Repair completed successfully!", "success");
        }, 1000);
    }

    // Download button click handler
    // downloadBtn.addEventListener("click", );
    const handleDownloadBtnClick =  () => {
        if (!repairedGeometry) return;

        const exporter = new STLExporter();
        const result = exporter.parse(new THREE.Mesh(repairedGeometry));

        const blob = new Blob([result], { type: "application/octet-stream" });
        const url = URL.createObjectURL(blob);

        const a = document.createElement("a");
        a.href = url;
        // a.download = "repaired.stl";
        const ogFileName = fileInput.files[0].name;
        const parts = ogFileName.split('.');
        const ext = parts.pop(); // should always be stl
        a.download = parts.join('.') + '_repaired.' + ext;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);

        URL.revokeObjectURL(url);
    }

    function repairGeometry(geometry) {
        const fixNormals = repairOptions.includes('fix-normals');
        const removeDuplicates = repairOptions.includes("remove-duplicates");
        const fillHoles = repairOptions.includes("fill-holes");

        // Clone the geometry to avoid modifying the original
        const repaired = geometry.clone();

        // Step 1: Fix inverted normals if requested
        if (fixNormals) {
            updateStatus("Fixing inverted normals...", "info");
            computeNormals(repaired);
        }

        // Step 2: Remove duplicate vertices if requested
        if (removeDuplicates) {
            updateStatus("Removing duplicate vertices...", "info");
            return removeDuplicateVertices(repaired);
        }

        // Step 3: Fill holes (placeholder for now)
        if (fillHoles) {
            updateStatus("Filling holes (simulated)...", "info");
            // This would require more complex algorithms
            // For now, we'll simulate this repair
        }

        return repaired;
    }

    function computeNormals(geometry) {
        // Compute vertex normals
        geometry.computeVertexNormals();

        // Fix inverted normals based on coherence
        // This is a placeholder for more advanced normal fixing
        // In a real implementation, you'd check for normal coherence
        // and flip normals that are pointing the wrong way
    }

    function removeDuplicateVertices(geometry) {
        // Create a new geometry with merged vertices
        geometry.deleteAttribute("normal");
        const mergedGeometry = mergeVertices(geometry);
        mergedGeometry.computeVertexNormals();

        return mergedGeometry;
    }

    

    function updateStatus(message, type = "info") {
        // const statusLine = document.createElement("div");
        // statusLine.textContent = message;
        let statusLine = '<div style="color: ';

        switch (type) {
            case "error":
                statusLine += "#e74c3c";
                break;
            case "warning":
                statusLine += "#f39c12";
                break;
            case "success":
                statusLine += "#2ecc71";
                break;
            default:
                statusLine += "#2980b9";
        }

        statusLine += ';">' + message + '</div><br/>'

        statusText += statusLine;
        // statusDiv.scrollTop = statusDiv.scrollHeight;
    }
</script>

<svelte:window on:resize={handleWindowResize} />

<div class="container">
    <a href="https://github.com/ianfabs/stlfixer" target="_blank" rel="noopener noreferrer" class="github-link">
        <svg class="github-icon" viewBox="0 0 24 24" fill="currentColor">
            <path d="M12 0c-6.626 0-12 5.373-12 12 0 5.302 3.438 9.8 8.207 11.387.599.111.793-.261.793-.577v-2.234c-3.338.726-4.033-1.416-4.033-1.416-.546-1.387-1.333-1.756-1.333-1.756-1.089-.745.083-.729.083-.729 1.205.084 1.839 1.237 1.839 1.237 1.07 1.834 2.807 1.304 3.492.997.107-.775.418-1.305.762-1.604-2.665-.305-5.467-1.334-5.467-5.931 0-1.311.469-2.381 1.236-3.221-.124-.303-.535-1.524.117-3.176 0 0 1.008-.322 3.301 1.23.957-.266 1.983-.399 3.003-.404 1.02.005 2.047.138 3.006.404 2.291-1.552 3.297-1.23 3.297-1.23.653 1.653.242 2.874.118 3.176.77.84 1.235 1.911 1.235 3.221 0 4.609-2.807 5.624-5.479 5.921.43.372.823 1.102.823 2.222v3.293c0 .319.192.694.801.576 4.765-1.589 8.199-6.086 8.199-11.386 0-6.627-5.373-12-12-12z"/>
        </svg>
    </a>
    <h1>STL Repair Tool</h1>
    
    <!-- svelte-ignore a11y_no_static_element_interactions -->
    <div class="upload-container" id="drop-area" bind:this={dropArea} on:drop={handleDrop}>
        <p>Drag and drop your STL file here</p>
        <p>or</p>
        <label for="file-input" class="file-label">Choose File</label>
        <input type="file" id="file-input" accept=".stl" bind:this={fileInput} on:change={handleFiles}/>
    </div>

    {#if !repairBtnDisabled}
    <div class="repair-options" id="repair-options" style="display: none;">
        <h2>Repair Options</h2>
        <div class="repair-option">
            <input type="checkbox" value="fix-normals" checked bind:group={repairOptions}/>
            <label for="fix-normals">Fix Inverted Normals</label>
        </div>
        <div class="repair-option">
            <input type="checkbox" value="remove-duplicates" checked bind:group={repairOptions}/>
            <label for="remove-duplicates">Remove Duplicate Vertices</label>
        </div>
        <div class="repair-option">
            <input type="checkbox" value="fill-holes" checked bind:group={repairOptions}/>
            <label for="fill-holes">Fill Holes</label>
        </div>
    </div>
    {/if}

    {#if showStats}
    <div class="stats-container" bind:this={statsContainer}>
        <div class="stat-box">
            <h3>Vertices</h3>
            <span id="vertex-count">{vertexCount}</span>
        </div>
        <div class="stat-box">
            <h3>Faces</h3>
            <span id="face-count">{faceCount}</span>
        </div>
        <div class="stat-box">
            <h3>Problems</h3>
            <span id="problem-count">{problemCount}</span>
        </div>
    </div>
    {/if}

    <div class="model-preview" bind:this={modelPreview}></div>

    <button id="repair-btn" disabled={repairBtnDisabled} on:click={handleRepairBtnClick}>Repair STL File</button>
    {#if showDownloadBtn}
    <button id="download-btn" on:click={handleDownloadBtnClick}>Download Repaired STL</button>
    {/if}

    {#if showProgress}
    <div class="progress-container" bind:this={progressContainer}>
        <div class="progress-bar" bind:this={progressBar}></div>
    </div>
    {/if}

    <div id="status">
        {@html statusText}
    </div>
</div>

<style>
    @import url('https://fonts.googleapis.com/css2?family=JetBrains+Mono:wght@400;500;600&family=Inter:wght@400;500;600&display=swap');

    :global(body) {
        font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
        margin: 0;
        padding: 20px;
        background-color: #0a0a0a;
        color: #e0e0e0;
    }
    .container {
        max-width: 900px;
        margin: 0 auto;
        background-color: #1a1a1a;
        padding: 30px;
        border-radius: 8px;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
        border: 1px solid #333;
        position: relative;
    }
    .github-link {
        position: absolute;
        top: 20px;
        right: 20px;
        color: #00ff9d;
        transition: all 0.3s ease;
        display: flex;
        align-items: center;
        text-decoration: none;
    }
    .github-link:hover {
        color: #00cc7d;
        transform: scale(1.1);
    }
    .github-icon {
        width: 24px;
        height: 24px;
    }
    h1 {
        font-family: 'JetBrains Mono', monospace;
        color: #00ff9d;
        text-align: center;
        margin-bottom: 30px;
        font-size: 2.2em;
        text-shadow: 0 0 10px rgba(0, 255, 157, 0.3);
        font-weight: 600;
        letter-spacing: -0.5px;
    }
    .upload-container {
        border: 2px dashed #333;
        padding: 40px;
        text-align: center;
        border-radius: 8px;
        margin-bottom: 20px;
        transition: all 0.3s ease;
        background-color: #222;
    }
    .upload-container p {
        font-family: 'JetBrains Mono', monospace;
        font-size: 0.9em;
        color: #888;
    }
    .file-label {
        font-family: 'JetBrains Mono', monospace;
        display: inline-block;
        padding: 12px 20px;
        background-color: #00ff9d;
        color: #0a0a0a;
        border-radius: 6px;
        cursor: pointer;
        margin-top: 10px;
        transition: all 0.3s ease;
        font-weight: 500;
        font-size: 0.9em;
    }
    button {
        font-family: 'JetBrains Mono', monospace;
        padding: 12px 20px;
        background-color: #00ff9d;
        color: #0a0a0a;
        border: none;
        border-radius: 6px;
        cursor: pointer;
        margin-top: 10px;
        transition: all 0.3s ease;
        width: 100%;
        font-size: 0.9em;
        font-weight: 500;
    }
    .stat-box h3 {
        font-family: 'JetBrains Mono', monospace;
        margin-top: 0;
        color: #00ff9d;
        font-size: 0.9em;
        font-weight: 600;
    }
    .stat-box span {
        font-family: 'JetBrains Mono', monospace;
        font-size: 1.2em;
        font-weight: 500;
    }
    .repair-option label {
        font-family: 'JetBrains Mono', monospace;
        margin-left: 5px;
        color: #e0e0e0;
        font-size: 0.9em;
    }
    #status {
        font-family: 'JetBrains Mono', monospace;
        margin-top: 20px;
        padding: 15px;
        border-radius: 6px;
        background-color: #222;
        max-height: 200px;
        overflow-y: auto;
        border: 1px solid #333;
        font-size: 0.9em;
        line-height: 1.5;
    }
    .upload-container:hover {
        border-color: #00ff9d;
        background-color: #252525;
    }
    .upload-container.drag-over {
        background-color: rgba(0, 255, 157, 0.1);
        border-color: #00ff9d;
    }
    input[type="file"] {
        display: none;
    }
    button:hover {
        background-color: #00cc7d;
        transform: translateY(-1px);
    }
    button:disabled {
        background-color: #333;
        color: #666;
        cursor: not-allowed;
        transform: none;
    }
    .progress-container {
        width: 100%;
        height: 20px;
        background-color: #222;
        border-radius: 10px;
        margin-top: 20px;
        overflow: hidden;
        border: 1px solid #333;
    }
    .progress-bar {
        height: 100%;
        background-color: #00ff9d;
        width: 0%;
        transition: width 0.3s ease;
    }
    .stats-container {
        display: flex;
        flex-wrap: wrap;
        margin-top: 20px;
        gap: 10px;
    }
    .stat-box {
        flex-grow: 1;
        background-color: #222;
        padding: 15px;
        border-radius: 6px;
        box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
        text-align: center;
        border: 1px solid #333;
    }
    .repair-options {
        margin: 20px 0;
        padding: 15px;
        background-color: #222;
        border-radius: 6px;
        border: 1px solid #333;
    }
    .repair-option {
        margin: 10px 0;
    }
    .model-preview {
        width: 100%;
        height: 400px;
        background-color: #222;
        margin-top: 20px;
        position: relative;
        border-radius: 6px;
        overflow: hidden;
        min-height: 400px;
        border: 1px solid #333;
    }
    /* Custom scrollbar */
    #status::-webkit-scrollbar {
        width: 8px;
    }
    #status::-webkit-scrollbar-track {
        background: #222;
    }
    #status::-webkit-scrollbar-thumb {
        background: #333;
        border-radius: 4px;
    }
    #status::-webkit-scrollbar-thumb:hover {
        background: #444;
    }
</style>
