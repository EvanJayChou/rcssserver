# EmbeddedSimulator Interface

## Header

`src/embedded_simulator/embeddedsimulator.h`

## Main Type

```cpp
class EmbeddedSimulator {
public:
    bool init();
    bool setTeamName( Side side, const std::string & name );
    bool enablePlayer( Side side,
                       int unum,
                       bool goalie = false,
                       double version = 18.0 );
    void startMatch();
    EmbeddedGameState step( const std::vector< EmbeddedPlayerCommand > & commands );
    EmbeddedGameState snapshot() const;

    // Trainer-style teleports (work in any play mode), for per-episode resets.
    bool moveBall( double x, double y, double vx = 0.0, double vy = 0.0 );
    bool movePlayer( Side side, int unum, double x, double y, double body_angle );
    void setPlayMode( PlayMode pm );
};
```

## Per-episode reset / teleport

Player `(move x y)` commands are only honoured by the engine in
`before_kick_off`. For RL curricula that re-place the ball and robots every
episode while staying in `play_on`, use the trainer-style teleports, which
place objects directly regardless of play mode:

```python
sim.set_play_mode(rcssserver_embedded.PlayMode.PM_PlayOn)  # so placements persist
sim.move_player(side, unum, x, y, body_angle_radians)      # angle in RADIANS
sim.move_ball(ball_x, ball_y)                              # vx, vy default to 0
```

Note: `move_player` takes the body angle in **radians** (native engine unit),
and `EmbeddedPlayerState.body_angle` from `snapshot()` is also in radians —
callers working in degrees must convert at the boundary.

## Inputs

### Team setup

```cpp
setTeamName( LEFT, "LeftTeam" );
setTeamName( RIGHT, "RightTeam" );
```

### Player setup

```cpp
enablePlayer( LEFT, 1 );
enablePlayer( RIGHT, 1 );
enablePlayer( LEFT, 2, true ); // goalie
```

### Commands

```cpp
struct EmbeddedPlayerCommand {
    Side side;
    int unum;
    std::string command;
};
```

`command` uses raw RoboCup player command text.

Examples:

```cpp
"(dash 50)"
"(turn 30)"
"(kick 100 0)"
"(move 0 0)"
"(tackle 45)"
```

## Outputs

```cpp
struct EmbeddedBallState {
    PVector pos;
    PVector vel;
};

struct EmbeddedPlayerState {
    Side side;
    int unum;
    bool enabled;
    bool goalie;
    PVector pos;
    PVector vel;
    double body_angle;
    double neck_angle;
};

struct EmbeddedGameState {
    int time;
    int stoppage_time;
    PlayMode playmode;
    int score_left;
    int score_right;
    EmbeddedBallState ball;
    std::vector< EmbeddedPlayerState > players;
};
```

## Call Order

```cpp
EmbeddedSimulator sim;

sim.init();
sim.setTeamName( LEFT, "LeftTeam" );
sim.setTeamName( RIGHT, "RightTeam" );

sim.enablePlayer( LEFT, 1 );
sim.enablePlayer( RIGHT, 1 );

sim.startMatch();

EmbeddedGameState state = sim.step( {
    { LEFT, 1, "(dash 50)" },
    { RIGHT, 1, "(turn 30)" }
} );
```

## Rules

- Call `init()` first.
- Set both team names before enabling players.
- Enable players before `startMatch()`.
- One `step()` call advances one simulator cycle.
- Commands are applied in input order.
- `snapshot()` returns current state without stepping.
- `players` contains all 22 player slots.
- Disabled players remain in `players` with `enabled == false`.
