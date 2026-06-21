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
</style>
