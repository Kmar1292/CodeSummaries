# Introduction
During my time as a game developer intern at Prosper I.T. Consulting, I engaged in two live projects that each lasted for two weeks. In the first live project, I worked with a team to recreate several classic arcade games in Unity. I created a modern version of the classic game Asteroids with WASD input to control player movement. Over the course of this two week sprint, I gained valuable experience using Unity and C# to create a game nearly from scratch. This project also brought me experience in working with a team including how to communicate effectively with teammates and reaching out in order to problem-solve any roadblocks I or other teammates had. In the second live project, I worked with a team to prototype a game in Unreal Engine 5. In this project, I worked with several systems within Unreal Engine including blueprints, metasounds, Lumen, Nanite, and more. 

# Projects
* [Asteroids - Unity](#Asteroids)
* [Obstacle Course - Unreal Engine](#Obstacle-Course-Unreal)

## Asteroids - Unity
### Screen-wrap
I created a script that could be applied to any game object to give it screen-wrap functionality. The edges of the screen are used as the boundaries of the gameplay area in order to prevent issues arising if players resize their screen during gameplay.
```
void Update() {
    //Get screen position of object in pixels since we want to check if player is moving off the edge of the screen
    Vector3 screenPosPixel = Camera.main.WorldToScreenPoint(transform.position);

    //Get edges of screen in world units
    float rightScreenWorld = Camera.main.ScreenToWorldPoint(new Vector2(Screen.width, Screen.height)).x;
    float leftScreenWorld = Camera.main.ScreenToWorldPoint(new Vector2(0f, 0f)).x;
    float topScreenWorld = Camera.main.ScreenToWorldPoint(new Vector2(Screen.width, Screen.height)).y;
    float botScreenWorld = Camera.main.ScreenToWorldPoint(new Vector2(0f, 0f)).y;

    //Check if object position is moving off the screen AND if the direction of movement is heading off screen, then change object position to other side of screen
    if (screenPosPixel.x <= 0 && rbody.velocity.x < 0) transform.position = new Vector2(rightScreenWorld, transform.position.y);
    else if (screenPosPixel.x >= Screen.width && rbody.velocity.x > 0) transform.position = new Vector2(leftScreenWorld, transform.position.y);
    else if (screenPosPixel.y >= Screen.height && rbody.velocity.y > 0) transform.position = new Vector2(transform.position.x, botScreenWorld);
    else if (screenPosPixel.y <= 0 && rbody.velocity.y < 0) transform.position = new Vector2(transform.position.x, topScreenWorld);
}
```

### Player Aiming and Shooting
The ship is constantly orientated towards the mouse cursor and fires when the left mouse button is pressed.
```
void playerAimShoot() {
    //Get mouse position in world units
    Vector3 mousePosition = Camera.main.ScreenToWorldPoint(Input.mousePosition);
    //Get direction to look at by subtracting mouse position with player object's position
    Vector2 direction = new Vector2((mousePosition.x - transform.position.x), (mousePosition.y - transform.position.y));
    //Rotate the object to direction
    transform.up = direction;

    //Shoot if left mouse button is clicked
    if (Input.GetMouseButtonDown(0)) {
        //Instantiate projectile object at player's position and rotation
        Asteroids_Projectile bullet = Instantiate(this.projectilePrefab, this.transform.position, this.transform.rotation);
        //Shoot the projectile using the Project method in direction of mouse cursor
        bullet.Project(direction);
    }
}
```

### Player Death and Respawn
The player loses a life upon collision with an asteroid and respawns in the middle of the gameplay area with a brief window of invulnerability.
```
private void OnCollisionEnter2D(Collision2D collision) {
    if (collision.gameObject.CompareTag("Enemy")) {
        soundManager.Play(playerExplosionAudio);    //Play explosion audio when player dies
        playerLives--;
        Destroy(collision.gameObject);  //Destroy asteroid
        //Explosion at where player died
        gameManager.Explosion(this.transform.position, this.transform.rotation);
        PlayerRespawn();
    }
}

private void PlayerRespawn() {
    //If player still has lives, bring player back to middle + blinking effect + invulnerability
    if(playerLives > 0) {
        this.transform.position = new Vector2(0, 0);
        StartCoroutine(PlayerRespawnBlink());
    }
    else {  //if player has 0 lives, turn player object invisible and put it on layer to ignore collisions
        gameObject.layer = LayerMask.NameToLayer("IgnoreProjectile");
        spriteRenderer.enabled = false;
    }
}

//Respawn blink + invulnerability
IEnumerator PlayerRespawnBlink(int times = 3) {
    for(int i = 0; i < times; i++) {
        //Switch layer to ignore collisions, effectively making player invulnerable
        gameObject.layer = LayerMask.NameToLayer("IgnoreProjectile");
        //Blink by turning sprite renderer on and off
        spriteRenderer.enabled = true;
        yield return new WaitForSeconds(respawnBlinkTime);
        spriteRenderer.enabled = false;
        yield return new WaitForSeconds(respawnBlinkTime);
        //bring player back to Player layer so it is affected by collisions again
        gameObject.layer = LayerMask.NameToLayer("Player");
        spriteRenderer.enabled = true;
    }
}
```

### Asteroid Destruction and Split
Asteroids are destroyed if it collides with a projectile or boundary, and if they are destroyed by a projectile the asteroid splits into two asteroids with half the size of the original and random vectors. 
```
public void OnCollisionEnter2D(Collision2D collision) {
    if (collision.gameObject.CompareTag("Projectile")) {
        //Play asteroid explosion sound when destroyed by player projectile
        soundManager.Play(asteroidExplosionAudio);
        //Explosion effect
        gameManager.Explosion(this.transform.position, this.transform.rotation);
        //Add score based on size of asteroid destroyed
        if (this.size < 1.0f && this.size >= minSize) gameManager.playerScore += 5f;
        else if (this.size >= 1.0f && this.size < 1.5f) gameManager.playerScore += 10f;
        else if (this.size >= 1.5f) gameManager.playerScore += 15f;

        //Split asteroid into two smaller asteroids if original asteroid size is at least twice the size of min size
        if (this.size / 2 >= this.minSize) {
            SplitAsteroid(this.size, new Vector2(this.transform.position.x, this.transform.position.y));
            Destroy(this.gameObject);
        }
        else Destroy(this.gameObject);
    }
    else if (collision.gameObject.CompareTag("Boundary")) Destroy(this.gameObject);
}

//Create child asteroids when parent asteroid hit by player projectile
void SplitAsteroid(float parentSize, Vector2 position, int times = 2) {
    for (int i = 0; i < times; i++) {
        Asteroids_AsteroidController childAsteroid = Instantiate(asteroidPrefab, position, Quaternion.AngleAxis(Random.Range(0f, 360f), Vector3.forward));
        //Make child asteroid size half the parent size
        childAsteroid.size = parentSize / 2;
        float childAsteroidForce = Random.Range(minAsteroidForce, maxAsteroidForce);
        childAsteroid.rbody.AddForce(Random.insideUnitCircle * childAsteroidForce);
    }
}
```

### Asteroid Spawn
Asteroids are spawned at random locations outside of the player's screen with trajectories directed towards the gameplay area. Asteroids are only spawned if the current number of asteroids is less than a designated threshold.
```
void SpawnAsteroid() {
    //Get current number of asteroids in play
    int currentNumberOfAsteroids = asteroidCount.Count();

    //Only spawn asteroid if current # of asteroids in play doesn't exceed max
    if (currentNumberOfAsteroids < maxNumberOfAsteroids) {
        //choose random side to spawn asteroid. 0 is top, 1 is right, 2 is bot, 3 is left
        int spawnSide = Random.Range(0, 4);

        switch (spawnSide) {
            case 0: //Top side
                CreateAsteroid(LeftEdgeSpawn, RightEdgeSpawn, TopEdgeCamera, TopEdgeSpawn, randomVector(Mathf.PI, 2 * Mathf.PI));
                break;
            case 1: //Right side
                CreateAsteroid(RightEdgeCamera, RightEdgeSpawn, BotEdgeSpawn, TopEdgeSpawn, randomVector(Mathf.PI / 2, 3 / 2 * Mathf.PI));
                break;
            case 2: //Bot side
                CreateAsteroid(LeftEdgeSpawn, RightEdgeSpawn, BotEdgeSpawn, BotEdgeCamera, randomVector(0f, Mathf.PI));
                break;
            case 3: //Left side
                CreateAsteroid(LeftEdgeSpawn, LeftEdgeCamera, BotEdgeCamera, TopEdgeCamera, randomVector(-Mathf.PI / 2, Mathf.PI / 2));
                break;
            default:
                Debug.Log("Error spawning asteroid");
                break;
        }
    }
}

//Create random angle between a min angle and max angle for asteroid trajectory
private Vector2 randomVector(float minRadians, float maxRadians) {
    float random = Random.Range(minRadians, maxRadians);
    return new Vector2(Mathf.Cos(random), Mathf.Sin(random));
}

//Create asteroid somewhere outside play area with random trajectory towards play area
public void CreateAsteroid(float leftEdge, float rightEdge, float botEdge, float topEdge, Vector2 direction) {
    //Randomize rotation of asteroid
    Quaternion randomRotation = Quaternion.AngleAxis(Random.Range(0f, 360f), Vector3.forward);
    //Randomize force to be applied to asteroid
    float asteroidForce = Random.Range(minAsteroidForce, maxAsteroidForce);
    Asteroids_AsteroidController asteroid = Instantiate(asteroidPrefab, new Vector2(Random.Range(leftEdge, rightEdge), Random.Range(botEdge, topEdge)), randomRotation);
    //Randomize size of asteroid
    asteroid.size = Random.Range(asteroid.minSize, asteroid.maxSize);
    asteroid.rbody.AddForce(direction * asteroidForce);
}
```

## Obstacle Course - Unreal Engine
