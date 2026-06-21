<script lang="ts">
    import { collection, doc, getDocs, updateDoc } from "firebase/firestore";
    import { firestore } from "./lib/firebase";

    console.log("skibidi sigma");

    let sponsorData = getData();
    async function getData() {
        const sponsors = await getDocs(collection(firestore, "sponsors"));
        const data: [
            name: string,
            data: { id: string; parking_spots: number },
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

    let parkingInput: HTMLInputElement, leavingInput: HTMLInputElement;
    async function park() {
        const id = parkingInput.value;
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
        const id = parkingInput.value;
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
        updateDoc(doc(firestore, "sponsors", sponsor), { parking_spots });
        sponsorData = getData();
    }
</script>

<h1>Parking In:</h1>
<input bind:this={parkingInput} />
<button on:click={park}>Park!</button>

<h1>Leaving:</h1>
<input bind:this={leavingInput} />
<button on:click={leave}>Leave!</button>

<center>
    {#await sponsorData then sponsors}
        {#each sponsors as [sponsor, data]}
            <div id="card">
                <h1>{sponsor}</h1>
                <p>{data.id}: {data.parking_spots}</p>
            </div>
        {/each}
    {:catch e}
        <h1>{e}</h1>
    {/await}
</center>

<style>
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
