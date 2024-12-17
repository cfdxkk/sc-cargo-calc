<script setup lang="ts">
    import { ref, watch } from 'vue';

    /**
     *          长  宽  高
     * 1  scu = 1 * 1 * 1
     * 2  scu = 2 * 1 * 1
     * 4  scu = 2 * 2 * 1
     * 8  scu = 2 * 2 * 2
     * 16 scu = 4 * 2 * 2
     * 24 scu = 6 * 2 * 2
     * 32 scu = 8 * 2 * 2
     * 
     * 单个箱子的尺寸及长宽高
     */
    const BOX_SIZE = {
        1: { x: 1, y: 1, z: 1 },
        2: { x: 2, y: 1, z: 1 },
        4: { x: 2, y: 2, z: 1 },
        8: { x: 2, y: 2, z: 2 },
        16: { x: 4, y: 2, z: 2 },
        24: { x: 6, y: 2, z: 2 },
        32: { x: 8, y: 2, z: 2 },
    };

    type CargoGrid3D = { x: number, y: number, z: number }; // 一个 3D 货物网格由长宽高构成
    type ShipCargoGrid3D = CargoGrid3D[]; // 一个飞船可能有多个货物网格

    /**
     * 每个飞船拥有的货物网格尺寸列表
     */
    const shipCargoList = ref<Record<string, ShipCargoGrid3D>>({
        rsi_polaris: [{ x: 12, y:6, z:4 }, { x: 12, y: 6, z: 4 }], // 576 SCU
        cru_c2: [{ x: 15, y: 8, z: 4 }, { x: 9, y: 6, z: 4 }], // 696 SCU
    });

    const selectedShip = ref({ shipName: "rsi_polaris", shipCargoGrid: shipCargoList["rsi_polaris"] }); // 用户选中的飞船，默认是 polaris

    const cargo1ScuCount = ref(0); // 用户填写的 1SCU 货物箱子的数量
    const cargo2ScuCount = ref(0); // 用户填写的 2SCU 货物箱子的数量
    const cargo4ScuCount = ref(0); // 用户填写的 4SCU 货物箱子的数量
    const cargo8ScuCount = ref(0); // 用户填写的 8SCU 货物箱子的数量
    const cargo16ScuCount = ref(0); // 用户填写的 16SCU 货物箱子的数量
    const cargo24ScuCount = ref(0); // 用户填写的 24SCU 货物箱子的数量
    const cargo32ScuCount = ref(0); // 用户填写的 32SCU 货物箱子的数量
    
    /**
     * 计算用户选中的飞船的货物网格在最优情况下能否装下数量为n的不同尺寸的货物箱子
     * @param shipCargoGrid 
     * @param param1 
     */
    type ScuCountType = { cargo1ScuCount: number, cargo2ScuCount: number, cargo4ScuCount: number, cargo8ScuCount: number, cargo16ScuCount: number, cargo24ScuCount: number, cargo32ScuCount: number }
    function checkSize(shipCargoGrid: ShipCargoGrid3D, { cargo1ScuCount, cargo2ScuCount, cargo4ScuCount, cargo8ScuCount, cargo16ScuCount, cargo24ScuCount, cargo32ScuCount }: ScuCountType): boolean {
        const cargoList: Array<{ scu: number, count: number }> = [
            { scu: 32, count: cargo32ScuCount },
            { scu: 24, count: cargo24ScuCount },
            { scu: 16, count: cargo16ScuCount },
            { scu: 8, count: cargo8ScuCount },
            { scu: 4, count: cargo4ScuCount },
            { scu: 2, count: cargo2ScuCount },
            { scu: 1, count: cargo1ScuCount }
        ];

        let boxesToPlace: Array<{x:number,y:number,z:number,scu:number}> = [];
        for (let c of cargoList) {
            if (c.count > 0) {
                const s = BOX_SIZE[c.scu];
                for (let i = 0; i < c.count; i++) {
                    boxesToPlace.push({ x: s.x, y: s.y, z: s.z, scu: c.scu });
                }
            }
        }

        // 按体积从大到小排序
        boxesToPlace.sort((a,b)=> (b.x*b.y*b.z - a.x*a.y*a.z));

        type FreeSpace = {
            x: number, y: number, z: number,
            w: number, h: number, d: number
        };

        const compartments: FreeSpace[][] = shipCargoGrid.map(grid => {
            return [{
                x: 0, y: 0, z: 0,
                w: grid.x,
                h: grid.y,
                d: grid.z
            }];
        });

        function placeBoxInFreeSpace(freeSpaces: FreeSpace[], fsIndex: number, box: {x:number,y:number,z:number}) {
            const fs = freeSpaces[fsIndex];
            // 移除被使用的空间
            freeSpaces.splice(fsIndex, 1);

            // 分割剩余空间
            if (box.x < fs.w) {
                freeSpaces.push({
                    x: fs.x + box.x,
                    y: fs.y,
                    z: fs.z,
                    w: fs.w - box.x,
                    h: fs.h,
                    d: fs.d
                });
            }

            if (box.y < fs.h) {
                freeSpaces.push({
                    x: fs.x,
                    y: fs.y + box.y,
                    z: fs.z,
                    w: box.x,
                    h: fs.h - box.y,
                    d: fs.d
                });
            }

            if (box.z < fs.d) {
                freeSpaces.push({
                    x: fs.x,
                    y: fs.y,
                    z: fs.z + box.z,
                    w: box.x,
                    h: box.y,
                    d: fs.d - box.z
                });
            }
        }

        function findFittingSpaces(freeSpaces: FreeSpace[], box:{x:number,y:number,z:number}): number[] {
            const result: number[] = [];
            for (let i = 0; i < freeSpaces.length; i++) {
                const fs = freeSpaces[i];
                if (box.x <= fs.w && box.y <= fs.h && box.z <= fs.d) {
                    result.push(i);
                }
            }
            return result;
        }

        let attempts = 0;
        const MAX_ATTEMPTS = 200000;

        function placeBoxRec(i: number): boolean {
            if (i === boxesToPlace.length) return true; // 全部放置成功
            if (attempts > MAX_ATTEMPTS) return false;  // 尝试过多，避免卡死

            attempts++;
            const box = boxesToPlace[i];

            // 为当前箱子尝试两种方向: (x,y,z) 和 (y,x,z)
            const orientations = [
                {x: box.x, y: box.y, z: box.z}, 
                {x: box.y, y: box.x, z: box.z}
            ];

            for (const orientation of orientations) {
                // 尝试将当前箱子以该方向放入任意货舱
                for (const freeList of compartments) {
                    const fittingSpaces = findFittingSpaces(freeList, orientation);
                    // 根据剩余体积浪费排序
                    fittingSpaces.sort((a,b)=>{
                        const va = freeList[a].w*freeList[a].h*freeList[a].d - orientation.x*orientation.y*orientation.z;
                        const vb = freeList[b].w*freeList[b].h*freeList[b].d - orientation.x*orientation.y*orientation.z;
                        return va - vb;
                    });

                    for (let fsIndex of fittingSpaces) {
                        const backup = freeList.slice();
                        placeBoxInFreeSpace(freeList, fsIndex, orientation);
                        if (placeBoxRec(i+1)) return true;
                        // 回溯
                        freeList.splice(0, freeList.length, ...backup);
                    }
                }
            }

            return false; // 无法放置当前箱子
        }

        return placeBoxRec(0);
    }

    const canFitCargo = ref<string | boolean | undefined>(undefined);
    // 计算结果
    function statCalc() {
        if (
            cargo1ScuCount.value < 0
            || cargo2ScuCount.value < 0
            || cargo4ScuCount.value < 0
            || cargo8ScuCount.value < 0
            || cargo16ScuCount.value < 0
            || cargo24ScuCount.value < 0
            || cargo32ScuCount.value < 0
        ) {
            canFitCargo.value = "The quantity of cargo cannot be negative.";
            return;
        }

        canFitCargo.value = checkSize(selectedShip.value.shipCargoGrid, {
            cargo1ScuCount : cargo1ScuCount.value,
            cargo2ScuCount : cargo2ScuCount.value,
            cargo4ScuCount : cargo4ScuCount.value,
            cargo8ScuCount : cargo8ScuCount.value,
            cargo16ScuCount : cargo16ScuCount.value,
            cargo24ScuCount : cargo24ScuCount.value,
            cargo32ScuCount : cargo32ScuCount.value,
        })
    };

    watch([
        selectedShip,
        cargo1ScuCount,
        cargo2ScuCount,
        cargo4ScuCount,
        cargo8ScuCount,
        cargo16ScuCount,
        cargo24ScuCount,
        cargo32ScuCount,
    ], () => {
        canFitCargo.value = undefined;
    })
</script>

<template>
    <div class="introduce-part">
        <p>
            This component provides an interactive tool to calculate whether a selected Star Citizen ship’s cargo grid can accommodate a 
            specified set of cargo boxes. By choosing a ship and inputting the quantity of various sized cargo boxes (1, 2, 4, 8, 16, 24, 32 SCU), 
            the underlying algorithm attempts to find a suitable arrangement. 
        </p>
        <p>
            The algorithm used here follows a backtracking strategy combined with a greedy heuristic: it sorts boxes from largest to smallest 
            before trying to place them into the defined cargo grids. This approach can quickly find a packing solution for many scenarios 
            due to its simplicity and straightforward logic. However, it does not guarantee finding an optimal solution in all cases and may 
            fail to place cargo even when a more complex algorithm might succeed. Additionally, the solution’s performance can degrade with 
            large input sizes, as backtracking can become computationally expensive. Thus, while this method is relatively easy to implement 
            and works well for moderate problem sizes, it may struggle with highly complex arrangements or require extensive computation time 
            to verify all possibilities.
        </p>

        <p>Special thanks: <a href="https://robertsspaceindustries.com/community-hub/post/cargo-grid-reference-guide-vqkv5cQI8ZCLC" target="_blank" >Cargo Grid Reference Guide</a> (Update date: December 12, 2024 at 4:19 AM)</p>
    </div>
    
    <p>1. Select your ship.</p>
    <select id="options" v-model="selectedShip">
      <option v-for="(shipCargoGrid, shipName) in shipCargoList" :key="shipName" :value="{ shipName, shipCargoGrid }">
        {{ shipName }}
      </option>
    </select>
    <p>You selected the: {{ selectedShip.shipName }} with {{ selectedShip.shipCargoGrid }} ship cargo grid.</p>
    
    <br />
    <p>2. Choose your cargo size and count.</p>
    <div>1  SCU * <input type="number" v-model="cargo1ScuCount" /></div>
    <div>2  SCU * <input type="number" v-model="cargo2ScuCount" /></div>
    <div>4  SCU * <input type="number" v-model="cargo4ScuCount" /></div>
    <div>8  SCU * <input type="number" v-model="cargo8ScuCount" /></div>
    <div>16 SCU * <input type="number" v-model="cargo16ScuCount" /></div>
    <div>24 SCU * <input type="number" v-model="cargo24ScuCount" /></div>
    <div>32 SCU * <input type="number" v-model="cargo32ScuCount" /></div>

    <br />
    <p>3. Start Calc.</p>
    <button @click="statCalc">Start</button>

    <br />
    <p>4. Results.</p>
    <div v-if="canFitCargo !== undefined">
        <p v-if="typeof canFitCargo === 'string'">{{ canFitCargo }}</p>
        <p v-if="canFitCargo === true">{{ selectedShip.shipName }} can fit the cargo</p>
        <p v-if="canFitCargo === false" class="unable-text">{{ selectedShip.shipName }} can not fit the cargo</p>
    </div>
</template>

<style lang="css" scoped>
    .introduce-part {
        width: 100dvw;
        max-width: 1500px;
    }

    .unable-text {
        color: red;
        font-weight: bold;
    }
</style>
