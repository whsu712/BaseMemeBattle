# BaseMemeBattle
BaseMemeBattle.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract BaseMemeBattle is Ownable, ReentrancyGuard {

    uint256 public constant MIN_BET = 0.001 ether;
    uint256 public constant MAX_BET = 0.1 ether;
    uint256 public houseEdge = 500; // 5% house edge (500 / 10000)

    uint256 public totalBattles;
    uint256 public totalWagered;
    uint256 public totalPayouts;

    enum Meme { Pepe, Doge, Shiba, Wojak, Chad, Moon }

    struct Battle {
        address player1;
        address player2;
        uint256 betAmount;
        Meme choice1;
        Meme choice2;
        Meme winner;
        bool isFinished;
        uint256 timestamp;
    }

    mapping(uint256 => Battle) public battles;
    mapping(address => uint256) public playerWins;
    mapping(address => uint256) public playerBattles;
    mapping(address => uint256) public playerTotalWagered;

    event BattleCreated(uint256 indexed battleId, address indexed player1, uint256 betAmount, Meme choice1);
    event BattleJoined(uint256 indexed battleId, address indexed player2, Meme choice2);
    event BattleResolved(uint256 indexed battleId, address winner, uint256 payout, Meme winningMeme);

    constructor() Ownable(msg.sender) {}

    // Player 1 creates a battle
    function createBattle(Meme myChoice) external payable nonReentrant {
        require(msg.value >= MIN_BET && msg.value <= MAX_BET, "Bet amount out of range");
        require(myChoice >= Meme.Pepe && myChoice <= Meme.Moon, "Invalid meme choice");

        totalBattles++;
        battles[totalBattles] = Battle({
            player1: msg.sender,
            player2: address(0),
            betAmount: msg.value,
            choice1: myChoice,
            choice2: Meme.Pepe,
            winner: Meme.Pepe,
            isFinished: false,
            timestamp: block.timestamp
        });

        playerTotalWagered[msg.sender] += msg.value;
        totalWagered += msg.value;

        emit BattleCreated(totalBattles, msg.sender, msg.value, myChoice);
    }

    // Player 2 joins the battle
    function joinBattle(uint256 battleId, Meme myChoice) external payable nonReentrant {
        Battle storage battle = battles[battleId];
        require(!battle.isFinished, "Battle already finished");
        require(battle.player2 == address(0), "Battle already has two players");
        require(msg.value == battle.betAmount, "Must match bet amount");
        require(myChoice >= Meme.Pepe && myChoice <= Meme.Moon, "Invalid meme choice");
        require(msg.sender != battle.player1, "Cannot join your own battle");

        battle.player2 = msg.sender;
        battle.choice2 = myChoice;
        battle.timestamp = block.timestamp;

        playerTotalWagered[msg.sender] += msg.value;
        totalWagered += msg.value;

        emit BattleJoined(battleId, msg.sender, myChoice);

        _resolveBattle(battleId);
    }

    function _resolveBattle(uint256 battleId) internal {
        Battle storage battle = battles[battleId];

        Meme winnerMeme = _determineWinner(battle.choice1, battle.choice2);

        address winnerAddress;
        uint256 payout = (battle.betAmount * 2 * (10000 - houseEdge)) / 10000;

        if (winnerMeme == battle.choice1) {
            winnerAddress = battle.player1;
            playerWins[battle.player1]++;
        } else if (winnerMeme == battle.choice2) {
            winnerAddress = battle.player2;
            playerWins[battle.player2]++;
        } else {
            // Draw - refund both players
            (bool s1, ) = payable(battle.player1).call{value: battle.betAmount}("");
            (bool s2, ) = payable(battle.player2).call{value: battle.betAmount}("");
            require(s1 && s2, "Refund failed");
            battle.isFinished = true;
            battle.winner = winnerMeme;
            emit BattleResolved(battleId, address(0), 0, winnerMeme);
            return;
        }

        require(address(this).balance >= payout, "Insufficient contract balance");
        (bool success, ) = payable(winnerAddress).call{value: payout}("");
        require(success, "Payout failed");

        battle.winner = winnerMeme;
        battle.isFinished = true;
        totalPayouts += payout;

        playerBattles[battle.player1]++;
        playerBattles[battle.player2]++;

        emit BattleResolved(battleId, winnerAddress, payout, winnerMeme);
    }

    // Meme battle logic with rock-paper-scissors style cycle
    function _determineWinner(Meme a, Meme b) internal pure returns (Meme) {
        if (a == b) return Meme.Pepe; // Draw

        if ((a == Meme.Pepe && b == Meme.Wojak) ||
            (a == Meme.Doge && b == Meme.Pepe) ||
            (a == Meme.Shiba && b == Meme.Doge) ||
            (a == Meme.Wojak && b == Meme.Shiba) ||
            (a == Meme.Chad && b == Meme.Wojak) ||
            (a == Meme.Moon && b == Meme.Chad)) {
            return a;
        }
        return b;
    }

    function setHouseEdge(uint256 newEdge) external onlyOwner {
        require(newEdge <= 1000, "House edge too high");
        houseEdge = newEdge;
    }

    function withdrawHouseFunds(uint256 amount) external onlyOwner nonReentrant {
        require(amount <= address(this).balance, "Insufficient balance");
        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "Withdraw failed");
    }

    function getPlayerStats(address player) external view returns (
        uint256 wins,
        uint256 battlesPlayed,
        uint256 totalWageredAmount
    ) {
        return (playerWins[player], playerBattles[player], playerTotalWagered[player]);
    }

    function getBattle(uint256 battleId) external view returns (Battle memory) {
        return battles[battleId];
    }

    receive() external payable {}
}
