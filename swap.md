### Pair
```
    function swap(uint256 amount0Out, uint256 amount1Out, address to, address referrer, bytes calldata data, bool antiMEV) external nonReentrant returns (bool) {
        require(amount0Out > 0 || amount1Out > 0, InsufficientOutputAmount());
        (uint112 _reserve0, uint112 _reserve1,) = getReserves();
        require(amount0Out < _reserve0 && amount1Out < _reserve1, InsufficientLiquidity());
        
        address _token0 = token0;
        address _token1 = token1;
        uint256 amount0In = IERC20(_token0).balanceOf(address(this)) - _reserve0;
        uint256 amount1In = IERC20(_token1).balanceOf(address(this)) - _reserve1;

        if (!IMEVGuard(MEVGuard).defend(antiMEV, _reserve0, _reserve1, amount0Out, amount1Out)) {
            address originTo = IMEVGuard(MEVGuard).originTo();
            require(originTo != address(0), EmptyOriginTo());
            if (amount0In != 0) _safeTransfer(_token0, originTo, amount0In);
            if (amount1In != 0) _safeTransfer(_token1, originTo, amount1In);
            emit SwapInterrupted(msg.sender, amount0In, amount1In, amount0Out, amount1Out, originTo);
            return false;
        }

        uint256 balance0;
        uint256 balance1;
        {
            require(to != _token0 && to != _token1, InvalidTo());

            if (amount0Out > 0) _safeTransfer(_token0, to, amount0Out);
            if (amount1Out > 0) _safeTransfer(_token1, to, amount1Out);
            if (data.length > 0) IOutrunAMMCallee(to).OutrunAMMCall(msg.sender, amount0Out, amount1Out, data);
            balance0 = IERC20(_token0).balanceOf(address(this));
            balance1 = IERC20(_token1).balanceOf(address(this));
        }

        unchecked {
            amount0In = balance0 > _reserve0 - amount0Out ? balance0 - (_reserve0 - amount0Out) : 0;
            amount1In = balance1 > _reserve1 - amount1Out ? balance1 - (_reserve1 - amount1Out) : 0;
        }
        require(amount0In > 0 || amount1In > 0, InsufficientInputAmount());
    
        uint256 rebateFee0;
        uint256 rebateFee1;
        uint256 protocolFee0;
        uint256 protocolFee1;
        uint256 _swapFeeRate = swapFeeRate;
        {
            uint256 realSwapFeeRate = antiMEV ? _realSwapFeeRate(_swapFeeRate) : _swapFeeRate;
            uint256 balance0Adjusted = balance0 * RATIO - amount0In * realSwapFeeRate;
            uint256 balance1Adjusted = balance1 * RATIO - amount1In * realSwapFeeRate;
            
            require(
                balance0Adjusted * balance1Adjusted >= uint256(_reserve0) * uint256(_reserve1) * RATIO ** 2,
                ProductKLoss()
            );

            address feeTo = _feeTo();
            (balance0, rebateFee0, protocolFee0) = _transferRebateAndProtocolFee(amount0In, balance0, realSwapFeeRate, _token0, referrer, feeTo);
            (balance1, rebateFee1, protocolFee1) = _transferRebateAndProtocolFee(amount1In, balance1, realSwapFeeRate, _token1, referrer, feeTo);
        }

        _update(balance0, balance1, _reserve0, _reserve1);

        {
            uint256 k = uint256(reserve0) * uint256(reserve1);
            // The market-making revenue from LPs that are proactively burned will be distributed to others
            uint256 actualSupply = totalSupply - proactivelyBurnedAmount;
            actualSupply = actualSupply == 0 ? 1 : actualSupply;
            feeGrowthX128 += (Math.sqrt(k) - Math.sqrt(kLast)) * FixedPoint128.Q128 / actualSupply;
            kLast = k;
        }

        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to, antiMEV);
        emit ProtocolFee(referrer, rebateFee0, rebateFee1, protocolFee0, protocolFee1);

        return true;
    }
```
### Factory
```
        function createPair(address tokenA, address tokenB) external override returns (address pair) {
            require(tokenA != tokenB, IdenticalAddresses());
    
            (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
            require(token0 != address(0), ZeroAddress());
            require(getPair[token0][token1] == address(0), PairExists()); // single check is sufficient
    
            bytes32 salt = keccak256(abi.encodePacked(token0, token1, swapFeeRate));
            pair = Clones.cloneDeterministic(pairImplementation, salt);
            IOutrunAMMPair(pair).initialize(token0, token1, MEVGuard, swapFeeRate);
            IMEVGuard(MEVGuard).setAntiFrontDefendBlockEdge(pair, block.number);
            
            getPair[token0][token1] = pair;
            getPair[token1][token0] = pair; // populate mapping in the reverse direction
            allPairs.push(pair);
    
            emit PairCreated(token0, token1, pair, allPairs.length);
        }
```
### Router
```
    function _swap(
        uint256[] memory amounts, 
        address[] memory path, 
        uint256[] memory feeRates, 
        address originTo, 
        address referrer,
        bool antiMEV
    ) internal returns (bool) {
        IMEVGuard(MEV_GUARD).setOriginTo(originTo);
        for (uint256 i; i < path.length - 1; i++) {
            (address input, address output) = (path[i], path[i + 1]);
            (address token0,) = OutrunAMMLibrary.sortTokens(input, output);
            uint256 amountOut = amounts[i + 1];
            (uint256 amount0Out, uint256 amount1Out) = input == token0 ? (uint256(0), amountOut) : (amountOut, uint256(0));
            address to = i < path.length - 2 ? OutrunAMMLibrary.pairFor(factories[feeRates[i + 1]], output, path[i + 2], feeRates[i + 1]) : originTo;
            if (!IOutrunAMMPair(OutrunAMMLibrary.pairFor(factories[feeRates[i]], input, output, feeRates[i])).swap(amount0Out, amount1Out, to, referrer, new bytes(0), antiMEV)) return false;
        }
        return true;
    }
```

