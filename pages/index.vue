<script setup lang="ts">
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
     * 计算单个箱子所占的长宽高
     */
    const boxDimensions = {
        1: [1, 1, 1],
        2: [2, 1, 1],
        4: [2, 2, 1],
        8: [2, 2, 2],
        16: [4, 2, 2],
        24: [6, 2, 2],
        32: [8, 2, 2],
    };

    type cargoGrid3D = [number, number, number];
    const shipCargoList = ref([
        {
            shipName: "polaris",
            cargoGrid: [[18, 4, 4], [18, 4, 4]],
        },
    ]);

    const selectedShip = ref("polaris");

    const cargo1ScuCount = ref(0);
    const cargo2ScuCount = ref(0);
    const cargo4ScuCount = ref(0);
    const cargo8ScuCount = ref(0);
    const cargo16ScuCount = ref(0);
    const cargo24ScuCount = ref(0);
    const cargo32ScuCount = ref(0);
    

    // 计算属性：检查是否能够装载给定数量的箱子
    const canFitCargo = computed(() => {
        const ship = shipCargoList.value.find((ship) => ship.shipName === selectedShip.value);
        if (!ship) return false;

        const boxCounts = {
            1: cargo1ScuCount.value,
            2: cargo2ScuCount.value,
            4: cargo4ScuCount.value,
            8: cargo8ScuCount.value,
            16: cargo16ScuCount.value,
            24: cargo24ScuCount.value,
            32: cargo32ScuCount.value,
        };

        const cargoGrids = ship.cargoGrid.map(([length, width, height]) => ({
            length,
            width,
            height,
            remainingSpaces: [{ x: 0, y: 0, z: 0, length, width, height }],
        }));

        for (const [scu, count] of Object.entries(boxCounts)) {
            const [boxLength, boxWidth, boxHeight] = boxDimensions[scu];

            for (let i = 0; i < count; i++) {
                let placed = false;

                for (const grid of cargoGrids) {
                    for (let j = 0; j < grid.remainingSpaces.length; j++) {
                        const space = grid.remainingSpaces[j];

                        // 检查箱子是否能放入当前剩余空间
                        if (
                            space.length >= boxLength &&
                            space.width >= boxWidth &&
                            space.height >= boxHeight
                        ) {
                            // 分割剩余空间
                            grid.remainingSpaces.splice(j, 1); // 移除当前使用的空间
                            grid.remainingSpaces.push(
                                { x: space.x + boxLength, y: space.y, z: space.z, length: space.length - boxLength, width: space.width, height: space.height }, // 剩余 X 方向空间
                                { x: space.x, y: space.y + boxWidth, z: space.z, length: boxLength, width: space.width - boxWidth, height: space.height }, // 剩余 Y 方向空间
                                { x: space.x, y: space.y, z: space.z + boxHeight, length: boxLength, width: boxWidth, height: space.height - boxHeight } // 剩余 Z 方向空间
                            );
                            placed = true;
                            break;
                        }
                    }

                    if (placed) break;
                }

                if (!placed) return false; // 如果当前箱子无法放置，直接返回 false
            }
        }

        return true; // 所有箱子均放置成功
    });

</script>

<template>
    <p>目前计算结果有误，正在优化算法。</p>
    <p>
        A Star Citizen cargo capacity calculator based on a greedy algorithm, used to calculate whether your ship can hold a certain amount of cargo.
    </p>
    <p>1. Select your ship.</p>
    <select id="options" v-model="selectedShip">
      <option v-for="shipCargo in shipCargoList" :key="shipCargo.shipName" :value="shipCargo.shipName">
        {{ shipCargo.shipName }}
      </option>
    </select>
    <br />
    <p>2. Choose your cargo size and count.</p>
    <div>1  SCU * <input type="number" v-model="cargo1ScuCount" /></div>
    <div>2  SCU * <input type="number" v-model="cargo2ScuCount" /></div>
    <div>4  SCU * <input type="number" v-model="cargo4ScuCount" /></div>
    <div>8  SCU * <input type="number" v-model="cargo8ScuCount" /></div>
    <div>16 SCU * <input type="number" v-model="cargo16ScuCount" /></div>
    <div>24 SCU * <input type="number" v-model="cargo24ScuCount" /></div>
    <div>32 SCU * <input type="number" v-model="cargo32ScuCount" /></div>

    
    <p>3. Results.</p>
    <p v-if="canFitCargo">{{ selectedShip.shipName }} can fit the cargo</p>
    <p v-else>{{ selectedShip.shipName }} can not fit the cargo</p>

    
</template>