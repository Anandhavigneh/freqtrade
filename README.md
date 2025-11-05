import numpy as np
import pandas as pd
from freqtrade.strategy import IStrategy
from pandas import DataFrame

class LSMAStrategy(IStrategy):
    # Strategy parameters
    lsma_length = 33
    lsma_offset = 33

    timeframe = '1h'

    minimal_roi = {
        "0": 0.02,  # 2% profit at any time
    }
    
    # Enable trailing stoploss
    trailing_stop = True
    trailing_stop_positive = 0.005  # 0.5% in profit before trailing activates
    trailing_stop_positive_offset = 0.01  # 1% offset from peak
    trailing_only_offset_is_reached = True

    stoploss = -0.01  # -1% Stoploss

    def lsma(self, prices, length):
        def _linreg(params):
            x = np.arange(len(params))
            slope, intercept = np.polyfit(x, params, 1)
            return intercept + slope * (length - 1)
        return prices.rolling(window=length).apply(_linreg, raw=True)

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # Calculate LSMA for the main pair
        dataframe['LSMA'] = self.lsma(dataframe['close'], self.lsma_length).shift(self.lsma_offset)
        return dataframe

    def populate_buy_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[
            (dataframe['LSMA'] < dataframe['low']) &
            ((dataframe['low'] - dataframe['LSMA']) / dataframe['low'] > 0.005),
            'buy'
        ] = 1

        return dataframe

    def populate_sell_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        return dataframe
