---
layout: post
title: "Screen wrapping with collision detection in Godot 4"
date: 2024-08-04 09:36:15 +0100
categories: game-dev godot
---

Recently, I wanted to achieve screen wrapping in Godot 4 that also checks for collisions first. I found a few solutions for screen wrapping but they all tend to break the golden rule of changing the `GlobalPosition` of a `PhysicsBody` which breaks collision detection, like this one on [kidscancode.org](https://kidscancode.org/godot_recipes/4.x/2d/screen_wrap/).

### My Requirements

When the player moves off the X axis of the screen, they should appear on the other side.

![Screen wrap requirement 1](/assets/screenwrap1.webp)

Unless there is an object on the other side blocking the player moving there.

![Screen wrap requirement 2](/assets/screenwrap2.webp)

### My Solution

To achieve this, I added 2 `RayCast2d` on each side of the screen that follow the player on the Y Axis. To visualise this, I've turned on `Visible Collision Paths` in the debug menu which shows the RayCasts as small blue arrows like in the demo below. They can then be used them to detect if there's space to move on the other side of the screen.

<video width="480" autoplay muted loop>
  <source src="/assets/solution-demo.webm" type="video/webm">
  Your browser does not support the video tag.
</video>

The RayCasts are positioned at the top and bottom of the player collision box to make sure that the player fits in the unblocked space. The `TargetPosition` of the RayCasts are calculated as half the width of the player collision box to make sure that half of the character can move over.

### Talk is cheap. Show me the code.

First, we want to add 4 `RayCast2d` nodes under the Player Scene in Godot and check `hit_from_inside` for each. It doesn't matter too much on their configuration in the editor as we will be changing them programmatically.

![Godot editor](/assets/editor.webp)

Next, in our Player script we add the following. This is in C# but I'm sure a certain AI ChatBot can convert it to GDScript if that's what you're using.

{% highlight c# %}
public partial class Player : CharacterBody2D
{

    private RayCast2D ScreenWrapRayCastTopLeft;
    private RayCast2D ScreenWrapRayCastTopRight;
    private RayCast2D ScreenWrapRayCastBottomLeft;
    private RayCast2D ScreenWrapRayCastBottomRight;

    private Vector2 playerCollisionShape;
    private Vector2 viewportSize;

    public override void _Ready()
    {
    	RectangleShape2D playerRectangleShape = collisionShape.Shape as RectangleShape2D;
    	playerCollisionShape = playerRectangleShape.Size;

    	ScreenWrapRayCastTopLeft = GetNode<RayCast2D>("ScreenWrapRayCastTopLeft");
    	ScreenWrapRayCastTopRight = GetNode<RayCast2D>("ScreenWrapRayCastTopRight");
    	ScreenWrapRayCastBottomLeft = GetNode<RayCast2D>("ScreenWrapRayCastBottomLeft");
    	ScreenWrapRayCastBottomRight = GetNode<RayCast2D>("ScreenWrapRayCastBottomRight");

    	viewportSize = GetViewportRect().Size;
    }

    public override void _PhysicsProcess(double delta)
    {
    	HandleMovement(delta);
    	WrapScreen();
    }

    private void UpdateRayCastPositions()
    {
    	Vector2 playerPosition = GlobalPosition;

    	// Set RayCast positions to the edge of the screen and half the Y axis of the player collision box
    	ScreenWrapRayCastTopLeft.GlobalPosition = new Vector2(0, playerPosition.Y - (playerCollisionShape.Y / 2));
    	ScreenWrapRayCastBottomLeft.GlobalPosition = new Vector2(0, playerPosition.Y + (playerCollisionShape.Y / 2));
    	ScreenWrapRayCastTopRight.GlobalPosition = new Vector2(viewportSize.X, playerPosition.Y - (playerCollisionShape.Y / 2));
    	ScreenWrapRayCastBottomRight.GlobalPosition = new Vector2(viewportSize.X, playerPosition.Y + (playerCollisionShape.Y / 2));

    	// Set RayCast directions to half of the player collision box X axis
    	ScreenWrapRayCastTopLeft.TargetPosition = new Vector2(playerCollisionShape.X / 2, 0);
    	ScreenWrapRayCastBottomLeft.TargetPosition = new Vector2(playerCollisionShape.X / 2, 0);
    	ScreenWrapRayCastTopRight.TargetPosition = new Vector2(-playerCollisionShape.X / 2, 0);
    	ScreenWrapRayCastBottomRight.TargetPosition = new Vector2(-playerCollisionShape.X / 2, 0);

    	// Update the raycasts
    	ScreenWrapRayCastTopLeft.ForceRaycastUpdate();
    	ScreenWrapRayCastBottomLeft.ForceRaycastUpdate();
    	ScreenWrapRayCastTopRight.ForceRaycastUpdate();
    	ScreenWrapRayCastBottomRight.ForceRaycastUpdate();
    }

    private void WrapScreen()
    {
    	UpdateRayCastPositions();

    	Vector2 position = GlobalPosition;

    	// We check both bottom and top RayCasts to make sure that the whole character and move in the space
    	bool canWrapLeft = !ScreenWrapRayCastTopLeft.IsColliding() && !ScreenWrapRayCastBottomLeft.IsColliding();
    	bool canWrapRight = !ScreenWrapRayCastTopRight.IsColliding() && !ScreenWrapRayCastBottomRight.IsColliding();

    	if (position.X < 0)
    	{
    		if (canWrapRight)
    		{
    			position.X = viewportSize.X;
    		}
    		else
    		{
    			position.X = 0;
    		}
    	}
    	else if (position.X > viewportSize.X)
    	{
    		if (canWrapLeft)
    		{
    			position.X = 0;
    		}
    		else
    		{
    			position.X = viewportSize.X;
    		}
    	}

    	GlobalPosition = position;
    }

}

{% endhighlight %}

### Conclusions

I tried a few solutions but this one worked the best for my use case. I hope this helps someone else who stumbles upon the same issue.
