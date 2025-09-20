---
layout: post
title: "Super Benji Post Mortem - js13k 2025"
date: 2025-09-19 05:04:15 +0100
categories: game-dev js13k
---

# Ideation

Growing up we had a black cat called Benji. Upon hearing the theme for this years js13k and paired with my current Snes addiction, I thought it would be cool if he could live forever on the internet as a Starfox character. Enter: Super Benji.

Being my 2nd js13k entry, I looked at some examples from the previous year to see how to do better sprites and animations. I ended up using [Those Final Seconds](https://js13kgames.com/2024/games/those-final-seconds) as a great example for spitesheet management, applying colour palettes and tilt shifting pixels for animation.

When first thinking about the core game loop, I wanted there to be satisfying clearing while looking forward to the next upgrade, similar to ARPGs like Path of Exile/Diablo and roguelitkes like Hades. I wanted to try to make the explosion visual and sound effects of defeating enemies as satisfying as possible.

I built the game mobile-first which explains the aspect-ratio, mouse movement and autoshooting. I did experiment with clicking the mouse on desktop for shooting, but I didn't want to change the controls depending on the device. It would have been nice to use the right click for special attacks or item abilities though.

# Music

I've never created sound with browser APIs so this was a good learning experience. I used a modified version of [TinyMusic](https://github.com/kevincennis/TinyMusic) that I converted to TypeScript and went through a few iterations of the soundtrack.

For the first version, I used this [Photoshop CS2 Keygen Chiptune music](https://www.youtube.com/watch?v=cYvjFOKOP7c&list=RDcYvjFOKOP7c&start_radio=1) as inspiration, but I couldn't get it to sound right and I got some feedback that it was repetitive and annoying.

<iframe width="560" height="315" src="https://www.youtube.com/embed/cYvjFOKOP7c?si=c_ITGloh9BHHUPjk" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I wanted to see how an atmospheric DnB track would feel with the space theme so I used some of my favourite tracks as inspiration. I did have some atmospheric pads in the background which reduced the repititiveness, but I had to remove them to save some bytes in a later cleanup.

<iframe width="560" height="315" src="https://www.youtube.com/embed/7Nuz_w3xLKM?si=W1c_sssToo_kl3-T" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# Artwork

I really liked how [Brewing Disaster](https://js13kgames.com/2024/games/brewing-disaster) used a GameBoy cover for the artwork, with my Snes collection I took that as inspiration to do something similar.

![Using Snes source material]({{ site.baseurl }}/assets/super-benji/benji-artwork.webp)

Using Starwing as inspiration for the theme of the game, I used [Bing AI Image creator](https://www.bing.com/images/create) to create the characters. I used "Starfox but with a Black Cat" as a prompt and I was amazed with the results. Then I pulled them into [Photopea](https://www.photopea.com) and used Snes box artwork for the rest. Here's some bonus artwork I didn't end up using:

![Using Snes source material]({{ site.baseurl }}/assets/super-benji/bonus-artwork.webp)

# Code

## player.explode()

With the size limitations of js13k, code reuse is really important. The explode mechanism is on the GameObject class that everything on-screen inherits from (Bullet, Player, Enemy, Boss), so it was really easy to implement and saved a lot of space.

It takes a random part of the sprite, then translates and rotates them outwards.

_GameObject.ts_

```
  explode(pieces: number = 8, pieceSize: number = EXPLOSION_PART_SIZE) {
    if (this.isExploding) return;
    this.velocity = { x: 0, y: 0 };
    this.isExploding = true;

    for (let i = 0; i < pieces; i++) {
      const speed = (Math.random() - 0.5) * EXPLOSION_SIZE; // px/sec instead of px/frame
      const angle = Math.random() * Math.PI * 2;

      this.explosionPieces.push({
        x: this.x + this.width / 2,
        y: this.y + this.height / 2,
        vx: Math.cos(angle) * speed,
        vy: Math.sin(angle) * speed,
        rot: Math.random() * Math.PI * 2,
        rotSpeed: (Math.random() - 0.5) * EXPLOSION_ROTATION_SPEED, // radians/sec
        alpha: 1,
        size: pieceSize,
        sx: Math.floor(Math.random() * (this.sprite.width - pieceSize)),
        sy: Math.floor(Math.random() * (this.sprite.height - pieceSize)),
      });
    }
  }

  drawExplosionParts(ctx: CanvasRenderingContext2D) {
    for (const p of this.explosionPieces) {
      ctx.save();
      ctx.globalAlpha = p.alpha;
      ctx.translate(p.x, p.y);
      ctx.rotate(p.rot);
      ctx.drawImage(
        this.sprite,
        p.sx,
        p.sy,
        p.size,
        p.size,
        -p.size / 2,
        -p.size / 2,
        p.size,
        p.size
      );
      ctx.restore();
    }
  }
```

_Enemy.ts_

```
  takeDamage(damage: number) {
    this.life -= damage;
    if (this.life <= 0) {
      this.explode();
      this.gameController.musicPlayer.playExplosionSound();
    }
  }
```

## Collision Detection

I made a static class for collision detection which handles everything, and could definitely be reused for future entries/projects:

_CollisionController.ts_

```
export class CollisionController {
  static isColliding(a: GameObject, b: GameObject): boolean {
    if (!a || !b) return false;
    return (
      a.x < b.x + (b.width || 0) &&
      a.x + (a.width || 0) > b.x &&
      a.y < b.y + (b.height || 0) &&
      a.y + (a.height || 0) > b.y
    );
  }

  static checkAll(
    objects: GameObject[],
    targets: GameObject[],
    callback: (a: GameObject, b: GameObject) => void
  ) {
    for (let i = objects.length - 1; i >= 0; i--) {
      const a = objects[i];
      if (!a.active) continue;

      for (let j = targets.length - 1; j >= 0; j--) {
        const b = targets[j];
        if (!b.active) continue;

        if (CollisionController.isColliding(a, b)) {
          callback(a, b);
          break; // stop checking this bullet against other enemies
        }
      }
    }
  }
}
```

_GameController.ts_

```
// Player Bullet and Enemy collision
CollisionController.checkAll(
  this.playerBulletPool.pool.filter((b) => b.active && !b.isExploding),
  this.enemies.filter((e) => !e.isExploding),
  (bulletObject, enemyObject) => {
    const enemy = enemyObject as Enemy;
    const bullet = bulletObject as Bullet;

    enemy.takeDamage(bullet.damage);
    bullet.explode(6, 2);
  }
);
```

# References

I used some references from other games as Easter Eggs, did you spot any?

- Mithril & Adamite armor upgrades from 2007 Runescape
- [Outer Wilds](https://store.steampowered.com/app/753640/Outer_Wilds/) zone name
- The sound effect when selecting an upgrade is based on Link opening a chest in Orcarina of Time
- Reverse Polarity Item - [Magnus' Ultimate](https://www.dota2.com/hero/magnus) from Dota2
- Jackal's super weapon is called the "Mega Drive" (Super Nintendo competitor)

I hope you enjoyed playing it!
