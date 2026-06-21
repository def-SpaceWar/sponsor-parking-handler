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

<!-- <nav>
    <h1>ID: <input bind:this={idInput} /></h1>
    <span></span>
    <button on:click={park}>Park!</button>
    <button on:click={leave}>Leave!</button>
</nav>

<center>
    {#await sponsorData then sponsors}
        {#each sponsors as [sponsor, data]}
            <div id="card">
                <h1>{sponsor}</h1>
                <p>ID: {data.id}</p>
                <p>Parking Spots: {data.parking_spots}</p>
                <p>License Plate: {data.license_plate}</p>
            </div>
        {/each}
    {:catch e}
        <h1>{e}</h1>
    {/await}
</center>

<style>
    nav {
        display: flex;

        span {
            flex-grow: 1;
        }

        button {
            font-size: 200%;
        }

        h1 {
            background-color: #555;

            input {
                height: 80%;
            }
        }
    }

    center {
        display: flex;
        flex-wrap: wrap;
    }

    #card {
        background-color: #aaa;
        flex: 30%;
        max-width: 30vw;
        margin: 1vw;
    }
</style> -->

<style>
    /* Global/Layout Reset variables */
    :global(body) {
        background-color: #0f172a;
        color: #f8fafc;
        font-family: "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
        margin: 0;
        padding: 2rem;
    }

    /* Navbar styling */
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

    /* Buttons */
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

    /* Responsive Main Grid */
    .grid-container {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
        gap: 1.5rem;
    }

    /* Cards */
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

    /* Statuses */
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
