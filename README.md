function การขอคืนเงิน และ ส่วนที่จัดการกับความล่าช้าที่ผู้เล่นไม่ครบทั้งสองคนเสียที
case 1 : มี player แค่คนเดียวแล้วรอนานจนขอยกเลิก จะทำการคืนเงินให้ player คนนั้นทั้งหมด
case 2 : มี 2 player แล้วไม่มีใครกด reveal และ ผ่านมาจนเกิน timelimit แล้วมีคนขอยกเลิกจะทำการแบ่ง reward ให้ทั้งคู่
case 3 : มี 2 player โดย player1 reveal แล้ว player2 ไม่ยอม reveal จะทำการโอน reward ให้ player1 
``` solidity
function cancelGame() public {
        if(numPlayer == 1 && timeUnit.elapsedSeconds() >= timeLimit){ //case1 
            address payable  account = payable(players[0]);
            account.transfer(reward);
        }
        else if (numPlayer == 2 && numReveal == 0 && timeUnit.elapsedSeconds() >= timeLimit){ //case2
            address payable account1 = payable(players[0]);
            address payable account2 = payable(players[1]);
            account1.transfer(reward/2);
            account2.transfer(reward/2);
        }
        else if (numPlayer == 2 && numReveal <= 1 && timeUnit.elapsedSeconds() >= timeLimit) { //case3
            address payable account = payable(msg.sender);
            address target;
            if(msg.sender == players[0])target = players[1];
            else target = players[0];
            (,,bool revealed) = commitReveal.commits(target);
            if(!revealed) {
                account.transfer(reward);
            }
        }
        else {
            require(false,"cannot refund");
        }
        reset();
    }

    function reset() private {
        reward = 0;
        for (uint i = 0; i < players.length; i++) {
            delete player_choice[players[i]];
            delete player_not_played[players[i]];
        }
        delete players;
        numPlayer = 0;
        numInput = 0;
        numReveal = 0;

    }
```
function ส่วนที่ทำการซ่อน choice และ commit
- ตรวจสอบว่ามีผู้เล่น 2 คนรึยัง
- ตรวจสอบว่าผู้เล่นคนนี้เคยตอบไปแล้วรึยัง
- การ commit
  - เริ่มจากการนำ choice ที่จะตอบไปต่อท้ายกับ random string แล้วทำการ hash โดยเก็บ orginal message ไว้ตรวจสอบทีหลัง
  - นำ hash มา commit
```solidity
function commit(bytes32 hash) public  {
        require(numPlayer == 2);
        require(player_not_played[msg.sender]);
        // require(choice == 0 || choice == 1 || choice == 2 || choice == 3 || choice == 4);
        // bytes32 hashInput = commitReveal.getHash(bytes32(choice));
        commitReveal.commit(hash,msg.sender);
        // player_choice[msg.sender] = choice;
        if (player_not_played[msg.sender]) {
            numInput ++;
            player_not_played[msg.sender] = false;
        }
        // if (numInput == 2) {
        //     _checkWinnerAndPay();
        // }
    }
```
function ส่วนทำการ reveal และนำ choice มาตัดสินผู้ชนะ
- นำ original message ที่เก็บไว้มาเข้า function โดย function จะเอา original message มา hash แล้ว check กับ message ที่ commit ไว้ว่าตรงกันรึเปล่า
- เมื่อทั้งคู่ reveal ครบแล้วจะตัดสินผลแพ้ชนะแล้วทำการแบ่ง reward
```solidity
function reveal(bytes32 hash) public {
        require(numPlayer == 2);
        require(numInput == 2);
        commitReveal.reveal(hash,msg.sender);
        uint choice = uint(uint8(hash[31]));
        player_choice[msg.sender] = choice;
        numReveal ++;
        if(numReveal == 2){
            _checkWinnerAndPay();
        }
        
    }
```
