# File structure

src/
- index.ts
    - processAction(action)
- engine.ts
- models.ts // Player, Card, etc live here
- utils/
    - players.ts
- games/
    peshaw/
    - effects.ts 
        // Has the list of effects
    - renderers.ts
        // Has a render(effect: Effect) -> string function that knows how to render each effect
    - engine.ts // Should just be pure functions that create state + effects, to make testing rly easy
- transports/
    - twilio.ts
    - telegram.ts
    - shell.ts
    - discord.ts
    - router.ts
- persistence/
    - redis.ts

# TODO: How do handle terminal users? What if two users have the same name?!
# TODO: how does rendering a particular effect work? Where does that happen? What's included in the effect?

# Server Code
## Start up:

## transports
Really tightly scoped to the functionality of the particular transport

- handleOutgoingAction(action: Action) // receives actions from router.ts, sends out actions via transport
- handleIncomingMessage(message: string) -> Action // receives message from the underlying transport, processes it into the correct action, hands it off to router.ts

## router.ts
- Should have a mapping of transport_name to transport
- processEffects(effects: Effect[]) 
    Receives effects from engine.ts, looks up the relevant player transports, then hands off to transports to send
- processAction(action: Action)
    Receives actions from a transport. Looks up the game, and then hands it off to that particular game engine



## Message Received Flow
- Generally, all of this should be async and decoupled. I.e. use IEFE
- Some transport gets a message
    - It looks up the player by the ID (either phone number, TG handle, etc - depends on the transport) using the correct function from utils/players.ts
    - Once it has the player, it goes and generates the Action (does this work? the action depends on which game you're using tho)
        - If it can't generate a valid action, it should return an UNKNOWN_ACTION effect and call processEffects()
    - It passes the action to router. That something then should:
        - Look up the game from the user
        - Pass the gameState + action to engine.ts
        engine.ts:
            - find the game's corresponding next function, which generates a new state + effects
            - It should persist the state, and then send the effects off to the router.ts
            - Router.ts then looks at each effect, looks up the player, and then for each transport the player is using, it goes and hands it off to the transport

# Utils
- players.ts
    - Has a function to look up active games for the player
        - findActiveGame(playerId: string) -> GameState | null
    - Has functions to look up a player by various IDs:
        - lookUpByPhone(number: string) -> Player
- game.ts

# Persistence

- Need a map of playerId -> Player 
- Need a map of gameId -> GameState


# Player 
{
    id: PlayerId (string, randomly generated at init?)
    name: string
    phone?: string // optional phone number
    telegramId?: string // or whatever the hell we use identify ppl on Telegram
    // etc, for the relevant transports
}

# Card
{
    suit: "hearts" | "diamonds"...
    rank: number (face cards get a rank to mark comparison easy)
}

# PlayedCard
{
    card: Card
    playerId: string
}

# Deck
{
    cards: Card[]
}

# Peshaw Game State

{
    game: "peshaw" (string literal)
    gameId: string (randomly generated?)
    playingState: "waiting" | "activate" | "finished" // Whether we're waiting for players to join, or we're started, or we're finished

    roundCardCount: number // How many cards are to be dealt this round

    playerIds: string[]
    dealer: PlayerId
    controllingPlayer: PlayerId // Who leads the current trick (plays first). Initially it's the player to the right of the dealer
    phase: "bidding" | "playing"
    trumpCard: Card

    currentTrick: PlayedCard[]
    previousTricks: {playerId: PlayedCard[][]} // Mapping of PlayerId to a list of list of PlayedCards

    currentPlayerId: PlayerId // id of the person whose turn it is
    currentRound: number
    hands: { playerId: Card[] } // mapping  of playerId -> Cards
    createdAt: Date
    updatedAt: Date
    bids: { playerId: PlayerId, bid: number, actual: number }[] // List of bids, actual tricks won, index in array representing turns
    scores: { playerId: PlayerId, score: number }[] // List of scores, index in array represents turns
}

# Functions

### Pheshaw Rules


function next(current: PeshawGameState, action: Action) -> { next: PeshawGameState, effects: Effect[] } {
    // Validate actions, return effects if needed
        // Is it the correct player's turn? (OUT_OF_TURN error)
        // Are we in the correct phase? (BIDDING_ENDED error, STILL_BIDDING error)
        // Validate the move is legal (ILLEGAL_MOVE error)
            // For peshaw, must always follow suit that is led (i.e. if you HAVE that suit in your hand, you MUST play it)
    // Create a next valid state
        // If bidding, add the bids to the bids state, advance the player
        // If currently playing, add the card to the currentTrick, and remove it from the player's hand, advance the player
        // If it's the end of the trick (eveyone played), see who won the trick and move it to the previousTricks state, advance the player
        // If everyone played all their cards, need to go to score it, and then move to game to the next state
    ...
}

# LEFT OFF:
- How do we handle errors? Right now, enumerating all the EffectTypes
- Work out scoring
- Work out game ending
- Really go thru and list what effects are issued and when
- How does the increasing/decreasing number of cards work?
- Should we have a LIMIT for the game length? Probably like, one week?


# EffectTypes (just a string Union)
- NotifyBiddingTurnOne // Updates ONE person telling them it's their time to bid
- NotifySomeoneBidAll // Updates EVERYONE that somebody just bid
- NotifyPlayingTurnOne // Updates ONE person telling them it's their time to play
- NotifyCardPlayedAll
- NotifyGameFinishedAll // Updates EVERYONE that the game is over, includes the final score
- NotifyTrickFinishedAll // Updates EVERYONE that a trick is over, say who won the trick
- NotifyRoundFinishedAll // Updates EVERYONE that a round is over, say who made their bid and who failed, give scoring recap
- NotifyScoringRecapAll
- NotifyError

# Effects: 


Basically should be a huge discriminated union
{
    effect: NotifiyOneBiddingTurn
    gameId 
    playerId
    gameState
}
{
    effect: NotifyAllSomeoneBid
    gameState
    playerId // bidding player
    bid
}
{
    effect: NotifyOneYourTurn
    gameState
    playerId
}
{
    effect: NotifyAllCardPlayed
    gameState
    playerId
    playedCard: Card
}
{
    effect: NotifyAllGameFinished
    gameState
}
{
    effect: NotifyAllTrickFinished
    gameState
    trick  // same type as earlier
    winner: PlayerId
}
{
    effect: NotifyAllRoundFinished
    gameState
    bids // same type as earlier
    scores
}
{
    effect: NotifyAllScoringRecap
    gameState
    scores
}
{ 
    effect: NotifyOneError
    error: PlayerError
    playerId
}

PlayerError (string union type)
- OUT_OF_TURN
- BIDDING_ENDED
- STILL_BIDDING
- ILLEGAL_MOVE
- UNKNOWN_ACTION

# Rendering effects

This section describes how certain effects are rendered into text.

### Notify One Bidding Turn

It shows text, your hand + trump, and then a diagram of who's bid so far, and then how many are bid out of total cards dealt

"""
It's your turn to bid!

Your Hand: [3❤️, 4♠️, K♦️　]
👑: 5♦️
-----
    Dealer
      ⌄  

Paul, Em, Larry, You, Mary, Gabby
            2     ^ 
       
Currently bid: 2 for 5

"""

### Notify All Someone Bid 
It shows who just bid, and then the state of the world
"""

Michael just bid 3! Now It's Mary's turn.

👑: 5♦️
-----
    Dealer
      ⌄  

Paul, Em, Larry, Michael, Mary, Gabby
            2      3       ^
       
Currently bid: 5 for 5

"""

### Notify One Your Turn
It tells you it's your turn, shows you your hand, and shows you what's been played so far.

"""
It's your turn to play!


Your Hand: [3❤️, 4♠️, K♦️　]
👑: 5♦️

----

This Trick 👻

    Dealer
      ⌄  

Paul, Em, Larry, You, Mary, Gabby
                  ^ 
       4♦️  5♦️

"""


🖤
### Notify All Card Played



# TODO: need to store histories somehow... maybe?

# Actions
- ListGames
- Bid
- Play Card
- Join (as either a new player, or create a player)
- Leave
- Start
- Scoring Recap


### Questions
- Can I get away w using Redis for this? I think I can...

### Ideas (for later)
- Should add an LLM layer that transforms text into known commands to let you play with natural language