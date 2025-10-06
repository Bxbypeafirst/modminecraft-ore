# modminecraft-ore
import { world, system, BlockLocation, ItemStack } from "@minecraft/server";

// สร้าง scoreboard สำหรับเก็บเลเวลนักขุด
system.runInterval(() => {
  const scoreboard = world.scoreboard;
  if (!scoreboard.getObjective("miner_level")) {
    scoreboard.addObjective("miner_level", "Miner Level");
  }
}, 40);

// รายชื่อแร่ที่ระบบตรวจจับ
const oreBlocks = [
  "minecraft:coal_ore",
  "minecraft:iron_ore",
  "minecraft:copper_ore",
  "minecraft:gold_ore",
  "minecraft:redstone_ore",
  "minecraft:lapis_ore",
  "minecraft:diamond_ore",
  "minecraft:emerald_ore",
  "minecraft:deepslate_coal_ore",
  "minecraft:deepslate_iron_ore",
  "minecraft:deepslate_copper_ore",
  "minecraft:deepslate_gold_ore",
  "minecraft:deepslate_redstone_ore",
  "minecraft:deepslate_lapis_ore",
  "minecraft:deepslate_diamond_ore",
  "minecraft:deepslate_emerald_ore",
  "minecraft:nether_quartz_ore",
  "minecraft:nether_gold_ore"
];

world.afterEvents.playerBreakBlock.subscribe(event => {
  const { block, player, dimension } = event;
  if (!oreBlocks.includes(block.typeId)) return;

  const heldItem = player.getComponent("minecraft:equippable").getEquipment("mainhand");
  if (!heldItem || !heldItem.typeId.includes("_pickaxe")) {
    player.sendMessage("⛏ ต้องถือ Pickaxe ถึงจะขุดแร่ด้วยพลัง Miner LV ได้!");
    return;
  }

  system.run(() => {
    const totalMined = mineOreCluster(dimension, block.location, oreBlocks);
    const level = getMinerLevel(player);
    const expGained = totalMined * (2 + level * 0.3);
    addMinerExp(player, expGained);

    reduceDurability(player, heldItem, totalMined, level);
    rewardPlayer(player, dimension, block.location, level);

    player.sendMessage(`💎 ขุดแร่สำเร็จ ${totalMined} ก้อน! +${Math.floor(expGained)} EXP`);
  });
});

// 🪓 ขุดต่อเนื่อง
function mineOreCluster(dimension, startLoc, oreBlocks) {
  const checked = new Set();
  const stack = [startLoc];
  let count = 0;

  while (stack.length > 0) {
    const loc = stack.pop();
    const key = `${loc.x},${loc.y},${loc.z}`;
    if (checked.has(key)) continue;
    checked.add(key);

    const block = dimension.getBlock(loc);
    if (!block || !oreBlocks.includes(block.typeId)) continue;

    block.setType("minecraft:air");
    dimension.spawnItem(new ItemStack(block.typeId.replace("_ore", ""), 1), loc);
    count++;

    const neighbors = [
      new BlockLocation(loc.x + 1, loc.y, loc.z),
      new BlockLocation(loc.x - 1, loc.y, loc.z),
      new BlockLocation(loc.x, loc.y + 1, loc.z),
      new BlockLocation(loc.x, loc.y - 1, loc.z),
      new BlockLocation(loc.x, loc.y, loc.z + 1),
      new BlockLocation(loc.x, loc.y, loc.z - 1)
    ];
    stack.push(...neighbors);
  }

  return count;
}

// ⚙️ ลด durability น้อยลงเมื่อเลเวลสูง
function reduceDurability(player, item, blocksMined, level) {
  const durability = item.getComponent("durability");
  if (!durability) return;

  // ลดการสึกหรอตามเลเวล (สูงสุดลดได้ 50%)
  const efficiency = Math.max(0.5, 1 - level * 0.02);
  const newDamage = durability.damage + Math.ceil(blocksMined * efficiency);
  durability.damage = Math.min(newDamage, durability.maxDurability);

  if (durability.damage >= durability.maxDurability) {
    player.sendMessage("❌ Pickaxe ของคุณพังแล้ว!");
    player.getComponent("minecraft:equippable").setEquipment("mainhand", undefined);
  } else {
    player.getComponent("minecraft:equippable").setEquipment("mainhand", item);
  }
}

// 📊 จัดการเลเวล Miner
function getMinerLevel(player) {
  const score = player.runCommand(`scoreboard players test "${player.name}" miner_level *`);
  const match = score.statusMessage.match(/-?\d+/);
  return match ? parseInt(match[0]) : 0;
}

function addMinerExp(player, amount) {
  const level = getMinerLevel(player);
  const newScore = level + Math.floor(amount / 50);
  player.runCommand(`scoreboard players set "${player.name}" miner_level ${newScore}`);

  if (newScore > level) {
    player.sendMessage(`✨ [LEVEL UP] คุณเป็น Miner LV.${newScore} แล้ว!`);
  }
}

// 🎁 ดรอปพิเศษตามเลเวล
function rewardPlayer(player, dimension, loc, level) {
  const chance = Math.random();
  let bonus = null;

  if (chance < 0.02 + level * 0.001) bonus = "minecraft:netherite_scrap"; // ของหายากมาก
  else if (chance < 0.05 + level * 0.002) bonus = "minecraft:diamond";
  else if (chance < 0.12 + level * 0.003) bonus = "minecraft:emerald";

  if (bonus) {
    dimension.spawnItem(new ItemStack(bonus, 1), loc);
    player.sendMessage(`🎁 คุณขุดเจอของหายาก: ${bonus.replace("minecraft:", "")}!`);
  }
}


