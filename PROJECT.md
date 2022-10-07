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


## Example Game Implementation

```elixir
defmodule Exp.Games do
  defmodule Game do
    alias Exp.Games.Player

    defmodule State do
      defstruct players: [], round: 1, choices: %{}
    end

    use GenServer

    def start(player_1_pid, player_2_pid) do
      GenServer.start(__MODULE__, [player_1_pid, player_2_pid])
    end

    def init(players) do
      {:ok, %State{players: players}}
    end

    def choose(game, player, choice) do
      GenServer.call(game, {:choose, player, choice})
    end

    def stats(game) do
      GenServer.call(game, :stats)
    end

    def handle_call({:choose, player, choice}, _from, state) do
      choices = make_choice(state.choices, player, choice)
      state = Map.put(state, :choices, choices)

      new_state =
        if round_complete?(choices) do
          handle_finished_game(state)
        else
          state
        end

      {:reply, new_state, new_state}
    end

    def handle_call(:stats, _from, state) do
      {:reply, %{round: state.round}, state}
    end

    defp round_complete?(choices) do
      player_count =
        choices
        |> Map.keys()
        |> length()

      player_count == 2
    end

    defp make_choice(choices, player, choice) do
      Map.put(choices, player, choice)
    end

    defp handle_finished_game(state) do
      results =
        state.players
        |> Enum.map(&Map.get(state.choices, &1))
        |> results()

      increment_player_results(state.players, results)

      %State{
        state
        | choices: %{},
          round: state.round + 1
      }
    end

    defp increment_player_results(players, results) do
      players
      |> Enum.with_index()
      |> Enum.each(fn {player, i} ->
        stat = results |> Enum.at(i) |> translate_stat()
        Player.increment_stat(player, stat)
      end)
    end

    defp translate_stat(:won), do: :wins
    defp translate_stat(:lost), do: :losses
    defp translate_stat(:tied), do: :ties

    def results([choice1, choice2]) do
      cond do
        {choice1, choice2} in [{:rock, :rock}, {:paper, :paper}, {:scissors, :scissors}] ->
          [:tied, :tied]

        {choice1, choice2} in [{:rock, :scissors}, {:paper, :rock}, {:scissors, :paper}] ->
          [:won, :lost]

        true ->
          [:lost, :won]
      end
    end
  end

  defmodule Player do
    use GenServer

    defmodule State do
      defstruct wins: 0, losses: 0, ties: 0
    end

    def new do
      start()
    end

    def increment_stat(player_pid, stat) do
      GenServer.call(player_pid, {:increment, stat})
    end

    def start do
      GenServer.start(__MODULE__, nil)
    end

    def init(_) do
      {:ok, %State{}}
    end

    def stats(player_pid) do
      GenServer.call(player_pid, :stats)
    end

    def handle_call(:stats, _from, state) do
      {:reply, Map.from_struct(state), state}
    end

    def handle_call({:increment, stat}, _from, state) do
      new_state = Map.put(state, stat, Map.get(state, stat) + 1)
      {:reply, new_state, new_state}
    end
  end
end
```
