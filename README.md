
extends Node3D

const BOT_COUNT = 20
const SPEED = 7.0
const BOT_SPEED = 2.8

var player: CharacterBody3D
var bots: Array[CharacterBody3D] = []
var hp := 100
var ammo := 60
var kills := 0
var shoot_timer := 0.0

func _ready():
    create_world()
    create_player()
    create_bots()

func create_world():
    var ground = StaticBody3D.new()
    var mesh = MeshInstance3D.new()
    var shape = CollisionShape3D.new()

    var plane = BoxMesh.new()
    plane.size = Vector3(200, 1, 200)
    mesh.mesh = plane
    mesh.position.y = -0.5

    var box = BoxShape3D.new()
    box.size = Vector3(200, 1, 200)
    shape.shape = box
    shape.position.y = -0.5

    ground.add_child(mesh)
    ground.add_child(shape)
    add_child(ground)

    var light = DirectionalLight3D.new()
    light.rotation_degrees = Vector3(-50, 30, 0)
    add_child(light)

    var environment = WorldEnvironment.new()
    var env = Environment.new()
    env.background_mode = Environment.BG_SKY
    env.ambient_light_source = Environment.AMBIENT_SOURCE_COLOR
    env.ambient_light_color = Color(0.7, 0.75, 0.8)
    environment.environment = env
    add_child(environment)

func make_character(color: Color) -> CharacterBody3D:
    var body = CharacterBody3D.new()
    var mesh = MeshInstance3D.new()
    var capsule = CapsuleMesh.new()
    capsule.height = 2
    capsule.radius = 0.4
    mesh.mesh = capsule
    mesh.material_override = StandardMaterial3D.new()
    mesh.material_override.albedo_color = color
    body.add_child(mesh)

    var collision = CollisionShape3D.new()
    var shape = CapsuleShape3D.new()
    shape.height = 2
    shape.radius = 0.4
    collision.shape = shape
    body.add_child(collision)
    return body

func create_player():
    player = make_character(Color.BLUE)
    player.position = Vector3(0, 1, 0)
    add_child(player)

    var camera = Camera3D.new()
    camera.position = Vector3(0, 4, 8)
    camera.rotation_degrees.x = -20
    player.add_child(camera)
    camera.current = true

func create_bots():
    for i in range(BOT_COUNT):
        var bot = make_character(Color.RED)
        bot.position = Vector3(
            randf_range(-60, 60), 1,
            randf_range(-60, 60)
        )
        bots.append(bot)
        add_child(bot)

func _physics_process(delta):
    if hp <= 0:
        return

    var direction = Vector3.ZERO

    if Input.is_key_pressed(KEY_W):
        direction.z -= 1
    if Input.is_key_pressed(KEY_S):
        direction.z += 1
    if Input.is_key_pressed(KEY_A):
        direction.x -= 1
    if Input.is_key_pressed(KEY_D):
        direction.x += 1

    direction = direction.normalized()
    player.velocity.x = direction.x * SPEED
    player.velocity.z = direction.z * SPEED
    player.velocity.y -= 20 * delta
    player.move_and_slide()

    shoot_timer -= delta

    for bot in bots:
        var target = player.global_position
        var chase = target - bot.global_position
        chase.y = 0

        if chase.length() > 1.5:
            bot.velocity.x = chase.normalized().x * BOT_SPEED
            bot.velocity.z = chase.normalized().z * BOT_SPEED
        else:
            bot.velocity.x = 0
            bot.velocity.z = 0

        bot.velocity.y -= 20 * delta
        bot.move_and_slide()

        if chase.length() < 15 and shoot_timer <= 0:
            hp -= 1

func shoot():
    if ammo <= 0:
        return

    ammo -= 1

    var closest = null
    var distance = 100000.0

    for bot in bots:
        var d = player.global_position.distance_to(bot.global_position)
        if d < distance:
            distance = d
            closest = bot

    if closest != null and distance < 40:
        bots.erase(closest)
        closest.queue_free()
        kills += 1

func _input(event):
    if event.is_action_pressed("ui_accept"):
        shoot()
