[gemini-code-1790697351793.java](https://github.com/user-attachments/files/32811520/gemini-code-1790697351793.java)
package com.nethrion.herobrine;

import org.bukkit.Bukkit;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.Sound;
import org.bukkit.World;
import org.bukkit.entity.ArmorStand;
import org.bukkit.entity.EntityType;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.entity.EntityDamageByEntityEvent;
import org.bukkit.event.player.PlayerInteractEvent;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.scheduler.BukkitRunnable;
import org.bukkit.util.Vector;

public class NethrionHerobrine extends JavaPlugin implements Listener {

    private boolean isEncounterActive = false;
    private ArmorStand herobrineBoss = null;
    private double bossHealth = 200.0;
    private boolean pushedAtHalfHealth = false;

    @Override
    public void onEnable() {
        getServer().getPluginManager().registerEvents(this, this);
        getLogger().info("Nethrion Herobrine End Overhaul 1.21.11 Loaded!");
    }

    @EventHandler
    public void onRitualTrigger(PlayerInteractEvent event) {
        Player player = event.getPlayer();
        // Simple ritual check: Right click Nether Star on Crying Obsidian in End
        if (event.getClickedBlock() != null 
                && event.getClickedBlock().getType() == Material.CRYING_OBSIDIAN 
                && player.getInventory().getItemInMainHand().getType() == Material.NETHER_STAR) {

            if (player.getWorld().getEnvironment() != World.Environment.THE_END) return;

            if (isEncounterActive) {
                player.sendMessage("§cHerobrine is already present in this realm!");
                return;
            }

            event.getClickedBlock().setType(Material.AIR);
            startRitualAndSummon(player, event.getClickedBlock().getLocation().add(0.5, 1, 0.5));
        }
    }

    private void startRitualAndSummon(Player player, Location loc) {
        isEncounterActive = true;
        bossHealth = 200.0;
        pushedAtHalfHealth = false;

        loc.getWorld().playSound(loc, Sound.ENTITY_WITHER_SPAWN, 1.0f, 0.5f);
        player.sendMessage("§4[Herobrine] §cYou shouldn't have done that...");

        // Spawn Native Entity (ArmorStand / Mannequin visual representation)
        herobrineBoss = (ArmorStand) loc.getWorld().spawnEntity(loc, EntityType.ARMOR_STAND);
        herobrineBoss.setCustomName("§4§lHEROBRINE");
        herobrineBoss.setCustomNameVisible(true);
        herobrineBoss.setGravity(false);
        herobrineBoss.setGlowing(true);

        // Start Encounter Loop
        new BukkitRunnable() {
            int ticks = 0;

            @Override
            public void run() {
                if (!isEncounterActive || herobrineBoss == null || herobrineBoss.isDead()) {
                    this.cancel();
                    return;
                }

                ticks++;

                // Target lock & Gaze (Death Stare)
                Player target = getNearestPlayer(herobrineBoss.getLocation(), 48);
                if (target != null) {
                    Location lookLoc = herobrineBoss.getLocation();
                    lookLoc.setDirection(target.getLocation().subtract(lookLoc).toVector());
                    herobrineBoss.teleport(lookLoc);

                    // Proximity Spatial Resistance (Infinity-Style Pressure)
                    double distance = herobrineBoss.getLocation().distance(target.getLocation());
                    if (distance < 6.0 && distance > 1.5) {
                        Vector pushAway = target.getLocation().toVector().subtract(herobrineBoss.getLocation().toVector()).normalize().multiply(0.15);
                        target.setVelocity(target.getVelocity().add(pushAway));
                    }
                }

                // 50% Health Phase Trigger: Almighty Push
                if (bossHealth <= 100.0 && !pushedAtHalfHealth) {
                    pushedAtHalfHealth = true;
                    executeAlmightyPush();
                }

                // Auto-death sequence
                if (bossHealth <= 0) {
                    executeDeathSequence();
                    this.cancel();
                }
            }
        }.runTaskTimer(this, 0L, 1L);
    }

    @EventHandler
    public void onBossDamage(EntityDamageByEntityEvent event) {
        if (herobrineBoss != null && event.getEntity().equals(herobrineBoss)) {
            bossHealth -= event.getFinalDamage();
            herobrineBoss.setCustomName("§4§lHEROBRINE §e[" + (int)bossHealth + "/200 HP]");

            if (bossHealth <= 0) {
                event.setCancelled(true);
                executeDeathSequence();
            }
        }
    }

    private void executeAlmightyPush() {
        if (herobrineBoss == null) return;
        World w = herobrineBoss.getWorld();
        w.playSound(herobrineBoss.getLocation(), Sound.ENTITY_GENERIC_EXPLODE, 2.0f, 0.5f);

        for (Player p : w.getPlayers()) {
            if (p.getLocation().distance(herobrineBoss.getLocation()) <= 30) {
                Vector pushVec = p.getLocation().toVector().subtract(herobrineBoss.getLocation().toVector()).normalize().multiply(2.5);
                pushVec.setY(0.5); // Safe vertical lift
                p.setVelocity(pushVec);
                p.sendMessage("§4[Herobrine] §lALMIGHTY PUSH!");
            }
        }
    }

    private void executeDeathSequence() {
        if (!isEncounterActive) return;
        isEncounterActive = false;

        if (herobrineBoss != null) {
            Location loc = herobrineBoss.getLocation();
            loc.getWorld().playSound(loc, Sound.UI_TOAST_CHALLENGE_COMPLETE, 1.0f, 1.0f);
            loc.getWorld().spawnEntity(loc, EntityType.NETHER_STAR); // Custom End Reward instead of Dragon Egg
            herobrineBoss.remove();
            herobrineBoss = null;
        }
        Bukkit.broadcastMessage("§6[The Kingdom SMP] §eHerobrine has been banished from the End!");
    }

    private Player getNearestPlayer(Location loc, double radius) {
        Player nearest = null;
        double nearestDist = radius * radius;
        for (Player p : loc.getWorld().getPlayers()) {
            double dist = p.getLocation().distanceSquared(loc);
            if (dist <= nearestDist) {
                nearestDist = dist;
                nearest = p;
            }
        }
        return nearest;
    }
}
