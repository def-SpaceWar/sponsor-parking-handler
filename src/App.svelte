<script lang="ts">
    import { collection, doc, getDocs, updateDoc } from "firebase/firestore";
    import { firestore } from "./lib/firebase";

    console.log("skibidi sigma");

    let sponsorData = getData();
    async function getData() {
        const sponsors = await getDocs(collection(firestore, "sponsors"));
        const data: [
            name: string,
            data: { id: string; parking_spots: number; license_plate: string },
        ][] = [];
        sponsors.forEach((s) => {
            const sData = s.data();
            if (sData.id && typeof sData.id == "string") {
                // @ts-ignore
                data.push([s.id, sData]);
            } else {
                console.error([s.id, sData]);
            }
        });
        return data;
    }

    let idInput: HTMLInputElement;

    // Track which sponsor is currently having their license plate edited
    let editingSponsorName: string | null = null;
    let editPlateValue = "";

    function startEditing(sponsorName: string, currentPlate: string) {
        editingSponsorName = sponsorName;
        editPlateValue = currentPlate || "";
    }

    function cancelEditing() {
        editingSponsorName = null;
        editPlateValue = "";
    }

    async function savePlate(sponsorName: string) {
        try {
            await updateDoc(doc(firestore, "sponsors", sponsorName), {
                license_plate: editPlateValue,
            });
            editingSponsorName = null;
            sponsorData = getData(); // Refresh data
        } catch (error) {
            alert("Failed to update license plate!");
            console.error(error);
        }
    }

    async function park() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsors = await sponsorData,
            sponsorM = sponsors.find(([_, data]) => data.id == id);
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
        updateDoc(doc(firestore, "sponsors", sponsor), { parking_spots });
        sponsorData = getData();
    }

    async function leave() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsors = await sponsorData,
            sponsorM = sponsors.find(([_, data]) => data.id == id);
        if (!sponsorM) {
            alert("SPONSOR NOT FOUND! " + id);
            return;
        }

        const [sponsor, parking_spots] = [
            sponsorM[0],
            sponsorM[1].parking_spots + 1,
        ];
        idInput.value = "";
        alert(sponsor + " has gained 1 parking spot back!");
        updateDoc(doc(firestore, "sponsors", sponsor), { parking_spots });
        sponsorData = getData();
    }
</script>

<nav>
    <div class="input-group">
        <label for="id-input">ID:</label>
        <input
            id="id-input"
            bind:this={idInput}
            placeholder="0000"
            maxlength="4"
        />
    </div>
    <div class="actions">
        <button class="btn btn-park" on:click={park}>Park!</button>
        <button class="btn btn-leave" on:click={leave}>Leave!</button>
    </div>
</nav>

<main class="grid-container">
    {#await sponsorData}
        <div class="loading">Loading sponsors...</div>
    {:then sponsors}
        {#each sponsors as [sponsor, data]}
            <div class="card">
                <h2>{sponsor}</h2>
                <div class="card-body">
                    <p>
                        <span class="label">ID:</span>
                        <span class="value code">{data.id}</span>
                    </p>
                    <p>
                        <span class="label">Parking Spots:</span>
                        <span
                            class="value badge {data.parking_spots === 0
                                ? 'empty'
                                : ''}"
                        >
                            {data.parking_spots}
                        </span>
                    </p>

                    <div class="plate-section">
                        <span class="label">License Plate:</span>
                        {#if editingSponsorName === sponsor}
                            <div class="edit-row">
                                <input
                                    class="inline-input"
                                    bind:value={editPlateValue}
                                />
                                <button
                                    class="icon-btn save"
                                    on:click={() => savePlate(sponsor)}
                                    >✓</button
                                >
                                <button
                                    class="icon-btn cancel"
                                    on:click={cancelEditing}>✕</button
                                >
                            </div>
                        {:else}
                            <div class="display-row">
                                <span class="value code"
                                    >{data.license_plate || "NONE"}</span
                                >
                                <button
                                    class="icon-btn edit"
                                    on:click={() =>
                                        startEditing(
                                            sponsor,
                                            data.license_plate,
                                        )}>✎</button
                                >
                            </div>
                        {/if}
                    </div>
                </div>
            </div>
        {/each}
    {:catch e}
        <div class="error">Error: {e.message}</div>
    {/await}
</main>

<!-- <script lang="ts">
    import { collection, doc, getDocs, updateDoc } from "firebase/firestore";
    import { firestore } from "./lib/firebase";

    console.log("skibidi sigma");

    let sponsorData = getData();
    async function getData() {
        const sponsors = await getDocs(collection(firestore, "sponsors"));
        const data: [
            name: string,
            data: { id: string; parking_spots: number; license_plate: string },
        ][] = [];
        sponsors.forEach((s) => {
            const sData = s.data();
            if (sData.id && typeof sData.id == "string") {
                // @ts-ignore
                data.push([s.id, sData]);
            } else {
                console.error([s.id, sData]);
            }
        });
        return data;
    }

    let idInput: HTMLInputElement, leavingInput: HTMLInputElement;

    async function park() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsors = await sponsorData,
            sponsorM = sponsors.find(([_, data]) => data.id == id);
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
        updateDoc(doc(firestore, "sponsors", sponsor), { parking_spots });
        sponsorData = getData();
    }

    async function leave() {
        const id = idInput.value;
        if (id.length !== 4) {
            alert("INVALID ID! " + id);
            return;
        }

        const sponsors = await sponsorData,
            sponsorM = sponsors.find(([_, data]) => data.id == id);
        if (!sponsorM) {
            alert("SPONSOR NOT FOUND! " + id);
            return;
        }

        const [sponsor, parking_spots] = [
            sponsorM[0],
            sponsorM[1].parking_spots + 1,
        ];
        idInput.value = "";
        alert(sponsor + " has gained 1 parking spot back!");
        updateDoc(doc(firestore, "sponsors", sponsor), { parking_spots });
        sponsorData = getData();
    }
</script>

<nav>
    <div class="input-group">
        <label for="id-input">ID:</label>
        <input
            id="id-input"
            bind:this={idInput}
            placeholder="0000"
            maxlength="4"
        />
    </div>
    <div class="actions">
        <button class="btn btn-park" on:click={park}>Park!</button>
        <button class="btn btn-leave" on:click={leave}>Leave!</button>
    </div>
</nav>

<main class="grid-container">
    {#await sponsorData}
        <div class="loading">Loading sponsors...</div>
    {:then sponsors}
        {#each sponsors as [sponsor, data]}
            <div class="card">
                <h2>{sponsor}</h2>
                <div class="card-body">
                    <p>
                        <span class="label">ID:</span>
                        <span class="value code">{data.id}</span>
                    </p>
                    <p>
                        <span class="label">Parking Spots:</span>
                        <span
                            class="value badge {data.parking_spots === 0
                                ? 'empty'
                                : ''}"
                        >
                            {data.parking_spots}
                        </span>
                    </p>
                    <p>
                        <span class="label">License Plate:</span>
                        <span class="value code"
                            >{data.license_plate || "N/A"}</span
                        >
                    </p>
                </div>
            </div>
        {/each}
    {:catch e}
        <div class="error">Error: {e.message}</div>
    {/await}
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
        gap: 1rem;
    }

    .input-group label {
        font-size: 1.25rem;
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
        border-color: #3b82f6;
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
            transform 0.2s,
            box-shadow 0.2s;
    }

    .card:hover {
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

    .badge {
        background: #065f46;
        color: #34d399;
        padding: 0.25rem 0.75rem;
        border-radius: 9999px;
        font-weight: bold;
    }

    .badge.empty {
        background: #7f1d1d;
        color: #f87171;
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
</style> -->

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
        gap: 1rem;
    }

    .input-group label {
        font-size: 1.25rem;
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
        border-color: #3b82f6;
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
            transform 0.2s,
            box-shadow 0.2s;
    }

    .card:hover {
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

    .card-body p,
    .plate-section {
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

    .badge {
        background: #065f46;
        color: #34d399;
        padding: 0.25rem 0.75rem;
        border-radius: 9999px;
        font-weight: bold;
    }

    .badge.empty {
        background: #7f1d1d;
        color: #f87171;
    }

    /* Inline Plate Editing styles */
    .display-row,
    .edit-row {
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }

    .inline-input {
        background: #0f172a;
        border: 1px solid #475569;
        border-radius: 4px;
        color: #fff;
        font-family: monospace;
        padding: 0.15rem 0.4rem;
        width: 100px;
    }

    .inline-input:focus {
        outline: none;
        border-color: #3b82f6;
    }

    .icon-btn {
        background: none;
        border: none;
        cursor: pointer;
        font-size: 0.9rem;
        padding: 0.2rem 0.4rem;
        border-radius: 4px;
        transition: background-color 0.2s;
    }

    .icon-btn.edit {
        color: #94a3b8;
    }
    .icon-btn.edit:hover {
        background: #334155;
        color: #fff;
    }
    .icon-btn.save {
        color: #10b981;
    }
    .icon-btn.save:hover {
        background: rgba(16, 185, 129, 0.2);
    }
    .icon-btn.cancel {
        color: #f87171;
    }
    .icon-btn.cancel:hover {
        background: rgba(248, 113, 113, 0.2);
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
