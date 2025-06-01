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


