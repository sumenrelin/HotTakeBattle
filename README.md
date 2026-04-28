# HotTakeBattle
HotTakeBattle.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

/// @title HotTakeBattle - A fun social hot take battle game on Base
/// @notice Players post hot takes and battle for likes and prizes
contract HotTakeBattle {

    uint256 public constant POST_FEE = 0.0002 ether;
    uint256 public constant LIKE_FEE = 0.0005 ether;
    uint256 public constant MAX_TAKE_LENGTH = 280;

    struct HotTake {
        string content;
        address author;
        uint256 likes;
        uint256 timestamp;
        bool claimed;
    }

    HotTake[] public allTakes;

    mapping(uint256 => mapping(address => bool)) public hasLiked;
    mapping(address => uint256) public totalLikesReceived;

    uint256 public totalPrizePool;

    event HotTakePosted(uint256 indexed takeId, address author, string content);
    event Liked(uint256 indexed takeId, address liker, uint256 newLikeCount);
    event RewardClaimed(uint256 indexed takeId, address author, uint256 amount);

    constructor() payable {
        if (msg.value > 0) {
            totalPrizePool = msg.value;
        }
    }

    /// @notice Post your hot take (opinion/meme)
    function postHotTake(string calldata _content) external payable {
        require(msg.value == POST_FEE, "Must send exact post fee");
        require(bytes(_content).length > 0 && bytes(_content).length <= MAX_TAKE_LENGTH, "Invalid content length");

        uint256 takeId = allTakes.length;

        allTakes.push(HotTake({
            content: _content,
            author: msg.sender,
            likes: 0,
            timestamp: block.timestamp,
            claimed: false
        }));

        totalPrizePool += POST_FEE;

        emit HotTakePosted(takeId, msg.sender, _content);
    }

    /// @notice Like a hot take (costs LIKE_FEE)
    function likeHotTake(uint256 _takeId) external payable {
        require(_takeId < allTakes.length, "Hot take does not exist");
        require(msg.value == LIKE_FEE, "Must send exact like fee");
        require(!hasLiked[_takeId][msg.sender], "You already liked this take");

        hasLiked[_takeId][msg.sender] = true;
        allTakes[_takeId].likes++;
        totalLikesReceived[allTakes[_takeId].author]++;

        totalPrizePool += LIKE_FEE;

        emit Liked(_takeId, msg.sender, allTakes[_takeId].likes);
    }

    /// @notice Claim rewards if your hot take is popular (top 3 get share of pool)
    function claimReward(uint256 _takeId) external {
        require(_takeId < allTakes.length, "Hot take does not exist");
        HotTake storage take = allTakes[_takeId];
        require(msg.sender == take.author, "Only author can claim");
        require(!take.claimed, "Already claimed");

        uint256 prize = 0;
        uint256 topLikes = getTopLikes();

        if (take.likes >= topLikes * 70 / 100) {  // Top tier
            prize = totalPrizePool * 40 / 100;
        } else if (take.likes >= topLikes * 40 / 100) {
            prize = totalPrizePool * 20 / 100;
        } else if (take.likes > 0) {
            prize = totalPrizePool * 5 / 100;
        }

        require(prize > 0, "No reward available");

        take.claimed = true;
        totalPrizePool -= prize;

        (bool success, ) = payable(msg.sender).call{value: prize}("");
        require(success, "Reward transfer failed");

        emit RewardClaimed(_takeId, msg.sender, prize);
    }

    /// @notice Get current top likes count
    function getTopLikes() public view returns (uint256) {
        uint256 maxLikes = 0;
        for (uint256 i = 0; i < allTakes.length; i++) {
            if (allTakes[i].likes > maxLikes) {
                maxLikes = allTakes[i].likes;
            }
        }
        return maxLikes;
    }

    /// @notice Get a specific hot take
    function getHotTake(uint256 _takeId) external view returns (HotTake memory) {
        require(_takeId < allTakes.length, "Hot take does not exist");
        return allTakes[_takeId];
    }

    /// @notice Get total number of hot takes
    function getTotalTakes() external view returns (uint256) {
        return allTakes.length;
    }

    /// @notice Emergency withdraw for contract owner (deployer)
    function emergencyWithdraw() external {
        require(msg.sender == tx.origin, "Only direct caller allowed");
        (bool success, ) = payable(msg.sender).call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }

    receive() external payable {
        totalPrizePool += msg.value;
    }
}
#[block:45298902 txIndex:158]from: 0x7C6...22450to: HotTakeBattle.(constructor)value: 0 weidata: 0x608...20033logs: 0hash: 0x2d9...5abd9
