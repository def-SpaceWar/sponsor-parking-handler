<script lang="ts">
    import {
        collection,
        doc,
        getDocs,
        updateDoc,
        Timestamp,
    } from "firebase/firestore";
    import { firestore } from "./lib/firebase";
    import { Html5QrcodeScanner } from "html5-qrcode";
    import { onDestroy, onMount } from "svelte";

    let rawSponsors: [
        name: string,
        data: {
            id: string;
            parking_spots: number;
            max_parking_spots: number;
            license_plate: string[];
            time: Timestamp | null;
        },
    ][] = [];

    let isLoading = true,
        errorMessage = "",
        currentIdValue = "",
        idInput: HTMLInputElement,
        newPlateInputs: Record<string, string> = {};

    let sortBy = "name";

    let scanner: Html5QrcodeScanner | null = null,
        isScanning = false;

    async function loadData() {
        try {
            isLoading = true;
            const sponsors = await getDocs(collection(firestore, "sponsors"));
            const data: typeof rawSponsors = [];

            sponsors.forEach((s) => {
                const sData = s.data();
                if (sData.id && typeof sData.id == "string") {
                    let plates: string[] = [];
                    if (Array.isArray(sData.license_plate)) {
                        plates = sData.license_plate;
                    } else if (
                        typeof sData.license_plate === "string" &&
                        sData.license_plate
                    ) {
                        plates = [sData.license_plate];
                    }

                    data.push([
                        s.id,
                        {
                            id: sData.id,
                            parking_spots: sData.parking_spots,
                            license_plate: plates,
                            max_parking_spots: sData.max_parking_spots || 0,
                            time: sData.time || null,
                        },
                    ]);
                } else {
                    console.error([s.id, sData]);
                }
            });
            rawSponsors = data;
        } catch (err: any) {
            errorMessage = err.message;
        } finally {
            isLoading = false;
        }
    }

    onMount(() => loadData());

    let arrangedSponsors: typeof rawSponsors;
    $: {
        let baseList = [...rawSponsors].sort((a, b) => {
            switch (sortBy) {
                case "name":
                    return a[0].localeCompare(b[0]);
                case "spots-desc":
                    return b[1].parking_spots - a[1].parking_spots;
                case "spots-asc":
                    return a[1].parking_spots - b[1].parking_spots;
                case "interacted":
                    const timeA = a[1].time ? a[1].time.toMillis() : 0;
                    const timeB = b[1].time ? b[1].time.toMillis() : 0;
                    return timeB - timeA;
            }
            return 0;
        });

        if (currentIdValue.length === 4) {
            const matchIndex = baseList.findIndex(
                ([_, data]) => data.id === currentIdValue,
            );
            if (matchIndex !== -1) {
                const [matchedSponsor] = baseList.splice(matchIndex, 1);
                baseList = [matchedSponsor, ...baseList];
            }
        }

        arrangedSponsors = baseList;
    }

    function handleIdInput(e: Event) {
        currentIdValue = (e.target as HTMLInputElement).value;
    }

    function toggleScanner() {
        isScanning = !isScanning;

        if (isScanning) {
            setTimeout(() => {
                scanner = new Html5QrcodeScanner(
                    "reader",
                    { fps: 10, qrbox: { width: 250, height: 250 } },
                    false,
                );
                scanner.render(onScanSuccess, onScanFailure);
            }, 50);
        } else {
            stopScanner();
        }
    }

    function onScanSuccess(decodedText: string) {
        const scannedId = decodedText.trim();

        if (scannedId.length === 4) {
            idInput.value = scannedId;
            currentIdValue = scannedId;
            stopScanner();
            alert(`Scanned ID successfully: ${scannedId}`);
        } else {
            console.warn(
                `Scanned code "${scannedId}" is not a valid 4-digit ID.`,
            );
        }
    }

    function onScanFailure(_error: any) {}

    function stopScanner() {
        if (scanner) {
            scanner
                .clear()
                .catch((err) => console.error("Failed to clear scanner:", err));
            scanner = null;
        }
        isScanning = false;
    }

    onDestroy(() => {
        if (scanner) scanner.clear();
    });

    function getParkingStatusClass(spots: number, max: number): string {
        if (spots === 0) return "empty";
        if (spots >= max) return "max";
        return "partial";
    }

    function formatTimestamp(timestamp: Timestamp | null): string {
        if (!timestamp) return "Never";
        return timestamp.toDate().toLocaleString([], {
            month: "short",
            day: "numeric",
            hour: "2-digit",
            minute: "2-digit",
        });
    }

    async function addPlate(sponsorName: string, currentPlates: string[]) {
        const newPlate = (newPlateInputs[sponsorName] || "")
            .trim()
            .toUpperCase();
        if (!newPlate) return;

        if (currentPlates.includes(newPlate)) {
            alert("This license plate is already added!");
            return;
        }

        const updatedPlates = [...currentPlates, newPlate];

        try {
            await updateDoc(doc(firestore, "sponsors", sponsorName), {
                license_plate: updatedPlates,
            });
            newPlateInputs[sponsorName] = "";
            await loadData();
        } catch (error) {
            alert("Failed to add license plate.");
            console.error(error);
        }
    }

    async function removePlate(
        sponsorName: string,
        plateToRemove: string,
        currentPlates: string[],
    ) {
        const updatedPlates = currentPlates.filter((p) => p !== plateToRemove);

        try {
            await updateDoc(doc(firestore, "sponsors", sponsorName), {
                license_plate: updatedPlates,
            });
            await loadData();
        } catch (error) {
            alert("Failed to remove license plate.");
            console.error(error);
        }
    }

    async function park() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsorM = rawSponsors.find(([_, data]) => data.id == id);
        if (!sponsorM) {
            alert("SPONSOR NOT FOUND! " + id);
            return;
        }

        const [sponsor, parking_spots] = [
            sponsorM[0],
            sponsorM[1].parking_spots - 1,
        ];
        if (parking_spots < 0) {
            alert("ALREADY USED ALL THEIR PARKING! " + sponsor);
            return;
        }
        idInput.value = "";
        currentIdValue = "";

        await updateDoc(doc(firestore, "sponsors", sponsor), {
            parking_spots,
            time: Timestamp.now(),
        });
        await loadData();
    }

    async function leave() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsorM = rawSponsors.find(([_, data]) => data.id == id);
        if (!sponsorM) {
            alert("SPONSOR NOT FOUND! " + id);
            return;
        }

        const [sponsor, parking_spots] = [
            sponsorM[0],
            sponsorM[1].parking_spots + 1,
        ];
        if (parking_spots > sponsorM[1].max_parking_spots) {
            alert("ALREADY HAS THEIR MAX PARKING SPOTS! " + sponsor);
            return;
        }
        idInput.value = "";
        currentIdValue = "";
        alert(sponsor + " has gained 1 parking spot back!");

        await updateDoc(doc(firestore, "sponsors", sponsor), {
            parking_spots,
            time: Timestamp.now(),
        });
        await loadData();
    }
</script>

<nav>
    <div class="input-group">
        <label for="id-input">ID:</label>
        <input
            id="id-input"
            bind:this={idInput}
            on:input={handleIdInput}
            placeholder="0000"
            maxlength="4"
        />
        <button
            class="btn btn-scan"
            class:active={isScanning}
            on:click={toggleScanner}
        >
            {isScanning ? "Close Cam" : "📷 Scan QR"}
        </button>

        <div class="sort-control">
            <label for="sort-select">Sort By:</label>
            <select id="sort-select" bind:value={sortBy}>
                <option value="name">Alphabetical (Name)</option>
                <option value="spots-desc"
                    >Spots Available (Most → Least)</option
                >
                <option value="spots-asc">Spots Available (Least → Most)</option
                >
                <option value="interacted">Last Interacted</option>
            </select>
        </div>
    </div>

    <div class="actions">
        <button class="btn btn-park" on:click={park}>Park!</button>
        <button class="btn btn-leave" on:click={leave}>Leave!</button>
    </div>
</nav>

{#if isScanning}
    <div class="scanner-wrapper">
        <div id="reader"></div>
    </div>
{/if}

<main class="grid-container">
    {#if isLoading}
        <div class="loading">Loading sponsors...</div>
    {:else if errorMessage}
        <div class="error">Error: {errorMessage}</div>
    {:else}
        {#each arrangedSponsors as [sponsor, data]}
            <div
                class="card"
                class:highlighted={currentIdValue === data.id &&
                    currentIdValue.length === 4}
            >
                <h2>{sponsor}</h2>
                <div class="card-body">
                    <p>
                        <span class="label">ID:</span>
                        <span class="value code">{data.id}</span>
                    </p>
                    <p>
                        <span class="label">Parking Spots:</span>
                        <span
                            class="value badge {getParkingStatusClass(
                                data.parking_spots,
                                data.max_parking_spots,
                            )}"
                        >
                            {data.parking_spots} / {data.max_parking_spots}
                        </span>
                    </p>
                    <p>
                        <span class="label">Last Interacted:</span>
                        <span class="value timestamp"
                            >{formatTimestamp(data.time)}</span
                        >
                    </p>

                    <div class="plates-wrapper">
                        <span class="label">License Plates:</span>

                        <div class="plates-list">
                            {#if data.license_plate.length === 0}
                                <span class="no-plates"
                                    >No plates registered</span
                                >
                            {/if}
                            {#each data.license_plate as plate}
                                <div class="plate-tag">
                                    <span class="code">{plate}</span>
                                    <button
                                        class="remove-btn"
                                        on:click={() =>
                                            removePlate(
                                                sponsor,
                                                plate,
                                                data.license_plate,
                                            )}>✕</button
                                    >
                                </div>
                            {/each}
                        </div>

                        <div class="add-plate-row">
                            <input
                                class="inline-input"
                                placeholder="NEW-123"
                                bind:value={newPlateInputs[sponsor]}
                                on:keydown={(e) =>
                                    e.key === "Enter" &&
                                    addPlate(sponsor, data.license_plate)}
                            />
                            <button
                                class="icon-btn add"
                                on:click={() =>
                                    addPlate(sponsor, data.license_plate)}
                                >+</button
                            >
                        </div>
                    </div>
                </div>
            </div>
        {/each}
    {/if}
</main>

<style>
    :global(body) {
        background-color: #0f172a;
        color: #f8fafc;
        font-family: "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
        margin: 0;
        padding: 2rem;
    }

    nav {
        display: flex;
        justify-content: space-between;
        align-items: center;
        flex-wrap: wrap;
        gap: 1.5rem;
        background: rgba(30, 41, 59, 0.7);
        backdrop-filter: blur(8px);
        padding: 1rem 2rem;
        border-radius: 12px;
        border: 1px solid rgba(255, 255, 255, 0.1);
        margin-bottom: 2rem;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
    }

    .input-group {
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        gap: 1rem;
    }

    .input-group label {
        font-size: 1.1rem;
        font-weight: 600;
        color: #94a3b8;
    }

    nav input {
        background: #0f172a;
        border: 2px solid #334155;
        border-radius: 6px;
        color: #fff;
        font-size: 1.25rem;
        padding: 0.5rem 1rem;
        width: 100px;
        text-align: center;
        font-family: monospace;
        transition: border-color 0.2s;
    }

    nav input:focus {
        outline: none;
        border-color: #eab308;
    }

    .sort-control {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        margin-left: 1rem;
        border-left: 1px solid #334155;
        padding-left: 1.5rem;
    }

    .sort-control select {
        background-color: #0f172a;
        color: #f8fafc;
        border: 2px solid #334155;
        border-radius: 6px;
        padding: 0.5rem 1rem;
        font-size: 1rem;
        cursor: pointer;
        outline: none;
        transition: border-color 0.2s;
    }

    .sort-control select:focus {
        border-color: #eab308;
    }

    .actions {
        display: flex;
        gap: 1rem;
    }

    .btn {
        font-size: 1.1rem;
        font-weight: 600;
        padding: 0.6rem 1.5rem;
        border: none;
        border-radius: 6px;
        cursor: pointer;
        transition:
            transform 0.1s,
            filter 0.2s;
    }

    .btn:active {
        transform: scale(0.97);
    }
    .btn-park {
        background-color: #10b981;
        color: #fff;
    }
    .btn-park:hover {
        filter: brightness(1.1);
    }
    .btn-leave {
        background-color: #3b82f6;
        color: #fff;
    }
    .btn-leave:hover {
        filter: brightness(1.1);
    }

    .btn-scan {
        background-color: #475569;
        color: #f8fafc;
        font-size: 1rem;
        padding: 0.5rem 1rem;
    }
    .btn-scan:hover {
        background-color: #64748b;
    }
    .btn-scan.active {
        background-color: #ef4444;
    }

    .scanner-wrapper {
        max-width: 500px;
        margin: 0 auto 2rem auto;
        background: #1e293b;
        padding: 1rem;
        border-radius: 12px;
        border: 1px solid #334155;
        box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.3);
    }

    :global(#reader) {
        border: none !important;
        color: #f8fafc !important;
    }
    :global(#reader button) {
        background-color: #3b82f6 !important;
        color: white !important;
        border: none !important;
        padding: 0.5rem 1rem !important;
        border-radius: 4px !important;
        cursor: pointer !important;
        margin-top: 10px !important;
    }
    :global(#reader select) {
        background-color: #0f172a !important;
        color: white !important;
        border: 1px solid #475569 !important;
        padding: 0.3rem !important;
        border-radius: 4px !important;
    }

    .grid-container {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
        gap: 1.5rem;
    }

    .card {
        background: rgba(30, 41, 59, 0.4);
        border: 1px solid rgba(255, 255, 255, 0.05);
        border-radius: 12px;
        padding: 1.5rem;
        box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        transition:
            transform 0.3s ease,
            border-color 0.3s ease,
            box-shadow 0.3s ease;
    }

    .card.highlighted {
        border-color: #eab308;
        box-shadow: 0 0 20px rgba(234, 179, 8, 0.4);
        background: rgba(234, 179, 8, 0.1);
        transform: translateY(-4px) scale(1.02);
        animation: pulseHighlight 2s infinite alternate;
    }

    @keyframes pulseHighlight {
        0% {
            box-shadow: 0 0 12px rgba(234, 179, 8, 0.2);
        }
        100% {
            box-shadow: 0 0 22px rgba(234, 179, 8, 0.5);
        }
    }

    .card:hover:not(.highlighted) {
        transform: translateY(-4px);
        box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4);
        border-color: rgba(255, 255, 255, 0.15);
    }

    .card h2 {
        margin: 0 0 1rem 0;
        font-size: 1.5rem;
        color: #f1f5f9;
        border-bottom: 1px solid #334155;
        padding-bottom: 0.5rem;
    }

    .card-body p {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin: 0.75rem 0;
    }

    .label {
        color: #94a3b8;
        font-size: 0.9rem;
    }
    .value {
        font-weight: 500;
    }
    .code {
        font-family: monospace;
        background: #1e293b;
        padding: 0.2rem 0.5rem;
        border-radius: 4px;
        color: #e2e8f0;
    }

    .timestamp {
        font-size: 0.9rem;
        color: #cbd5e1;
    }

    .badge {
        padding: 0.25rem 0.75rem;
        border-radius: 9999px;
        font-weight: bold;
    }
    .badge.max {
        background: #065f46;
        color: #34d399;
    }
    .badge.partial {
        background: #78350f;
        color: #fbbf24;
    }
    .badge.empty {
        background: #7f1d1d;
        color: #f87171;
    }

    .plates-wrapper {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        margin-top: 1rem;
        padding-top: 0.75rem;
        border-top: 1px dashed #334155;
    }

    .plates-list {
        display: flex;
        flex-wrap: wrap;
        gap: 0.4rem;
        margin: 0.5rem 0;
    }
    .plate-tag {
        display: inline-flex;
        align-items: center;
        background: #1e293b;
        border-radius: 4px;
        padding-right: 0.25rem;
        border: 1px solid #475569;
    }
    .plate-tag .code {
        background: none;
        padding: 0.2rem 0.4rem;
    }

    .remove-btn {
        background: none;
        border: none;
        color: #f87171;
        cursor: pointer;
        font-size: 0.75rem;
        padding: 0 0.2rem;
        border-radius: 2px;
    }
    .remove-btn:hover {
        background: rgba(248, 113, 113, 0.2);
    }
    .no-plates {
        font-size: 0.85rem;
        color: #64748b;
        font-style: italic;
    }

    .add-plate-row {
        display: flex;
        gap: 0.5rem;
        margin-top: 0.25rem;
    }
    .inline-input {
        flex-grow: 1;
        background: #0f172a;
        border: 1px solid #475569;
        border-radius: 4px;
        color: #fff;
        font-family: monospace;
        padding: 0.3rem 0.5rem;
        font-size: 0.9rem;
    }
    .inline-input:focus {
        outline: none;
        border-color: #eab308;
    }

    .icon-btn.add {
        background: #3b82f6;
        color: white;
        border: none;
        border-radius: 4px;
        padding: 0 0.75rem;
        font-weight: bold;
        cursor: pointer;
    }
    .icon-btn.add:hover {
        background: #2563eb;
    }

    .loading,
    .error {
        grid-column: 1 / -1;
        text-align: center;
        font-size: 1.25rem;
        color: #94a3b8;
        padding: 3rem;
    }
    .error {
        color: #f87171;
    }
</style>
