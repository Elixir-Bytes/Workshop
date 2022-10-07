## Link to mockup
https://www.figma.com/file/2zXcMAVlFGoiroHbbnLntt/RPS?node-id=1%3A19

**Pages**

- Landing
- Game

**Game states**

- Waiting for opponent
- Waiting for both choices
- Waiting for opponent choice
- Waiting for my choice
- (Optional) Counter
- Solution

**Extra features**

- Somehow copy link to page

**Design**

- Landing page can just be PageController.index
- Game page is LiveView
- GenServer holds all games states
- Game is a Struct
- Game Struct has:
	- users, random UUIDs generated on LiveView creation
	- choices
	- UUID of game is the ID part of the live_view's params
	- results `%{winner: user_id(), loser: user_id()}`
- GameServer
	- `create_game(game_id, user_id) :: game (or :ok)`
	- `choose(user_id, choice) :: {:complete, results()} | {:choice, {user_id, choice}`
	- NOTE: Deletes game after final choice
- PageIndex template
  - Generates random UUID to use as ephemeral Game ID and adds it to the live link
- LiveView (`GameLive.Index`)
	- creates game on mount from game id in params
	- `handle_event("choice", %{"choice" => "paper"}, socket)`
		- Sends choice to the `GameServer`
		- if game over, update socket with winning info
		- set game state to `:complete` so UI can update

	- In template have conditionals basted on game state or results that only show part of the UI in different states
	- You can create helper functions in the LiveView to render components, like the:
	```
	DU
	[ ] [ ] [ ]
	```
	or
	```
	GEGNER

    Waiting on user
	```
	parts
	- State updated after choosing should change the UI but we can stay in this `GameLive.Index`


**Interesting additions**

- Persist game states
	- Leaderboard (would need some kind of authentication, at the very least a username input on start page)
