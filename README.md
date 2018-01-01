# Bit-BaiBai

Bit-BaiBai is a combination of three tools that work together to automate the
buying and selling of crypto currencies.

1. **BaiBai-Trader** is the most important module. This where trading actually
   happens.
2. **BaiBai-Server** is a small server that reads BaiBai-Trader's log files and
   serves serves them up in JSON format via websockets.
3. **BaiBai-Watcher** is a minimalistic web frontend that plots BaiBai-Trader's
   activity in realtime using the data exposed by BaiBai-Server.

Each of these three processes runs independently from one another in its own
Docker container, and Docker Compose is used to tie them together and make them
operate in unison.

Bit-BaiBai was originally intended to be run on a Raspberry
Pi server, so care has been taken to ensure it runs on both x86 and arm7.

# Getting Started

1. Install docker and docker-compose
2. Clone the project and all it's subprojects
   `git clone --recursive -j8 git://github.com/foo/bar.git`
3. Enter the root directory and start docker-compose
   `docker-compose up`
4. In a web browser, navigate to `localhost:3000` to watch your bot

# Providing Credentials

By default Bit-BaiBai will run in practice mode. Buys and sells will be
simulated, but not actually executed. When you are ready to take the plunge,
provide credentials to the market you trade on and choose an algorithm to go
live.

#### Kraken

When using the Kraken authenticator, you must provide a keyfile. First, log into
Kraken and create a new api key. Next, create a new text file and copy the api
key to the first line of the new file and your api secret to the second line.
Make sure to keep this information secret!

Once you've created a key file, edit `main.py` to use the Kraken authenticator
with your personal key file.

```python
auth = KrakenAuthenticator('path/to/your/keyfile', 'XBT', 'USD')
```

# Writing Your Own Logic

Bit-BaiBai is built to be extended. Presently it only supports one algorithm and
one market, but adding more is simple.

#### Adding New Algorithms

You can add a new algorithm by writing your own subclass of `Algorithm` and
adding it to `baibai-trader/baibaitrader/Algorithms`. You can then swap your own
algorithm into `main.py`. See `Algorithm.py` for details about the arguments to
each method and what their return types should be.

```python
from .Algorithm import Algorithm

class MyAlgorith(Algorithm):

    def process_data(self, price_samples):
        super().process_data(price_samples)
        # accumulate prices here
        pass

    def check_should_buy(self):
        super().check_should_buy()
        # Define how to decide when to buy
        pass

    def check_should_sell(self):
        super().check_should_sell()
        # Define how to decide when to sell
        pass

    def determine_buy_volume(self, price, holdings, account_balance):
        super().determine_buy_volume(price, holdings, account_balance)
        # When buying, determine how many shares to buy
        pass

    def determine_sell_volume(self, price, holdings, account_balance):
        super().determine_sell_volume(price, holdings, account_balance)
        # When selling, determine how many shares to sell
        pass
```

#### Adding New Markets

Currently the only market supported is Kraken, but will we add other markets
if there is enough interest. Markets can be added the same way as algorithms --
by subclassing an abstract base class. After writing a custom subclass of
`Authenticator` and adding it to `baibai-trader/baibaitrader/Markets`,
substitute your new class into `main.py` to use it. See `Authenticator.py` for
details about the arguments to each method and what their return types should
be.

```python
from .Authenticator import Authenticator

class SomeMarket(Authenticator):

    def target_currency(self):
        # The currency being bought
        return "XRP"

    def price_currency(self):
        # The currency being spent
        return "USD"

    def get_current_price(self):
        super().get_current_price()
        # API call to get current price of asset
        pass

    def get_account_balance(self):
        super().get_account_balance()
        # API call to get current balance
        pass

    def get_holdings(self):
        super().get_holdings()
        # API call to get current holdings
        pass

    def buy(self, n_shares):
        super().buy(n_shares)
        # API call to buy shares
        pass

    def sell(self, n_shares):
        super().sell(n_shares)
        # API call to see shares
        pass
```

# Testing and Validation

In this context bugs have serious repurcussions, so all `Algorith` and `Market`
classes should have 100% unit test coverage. Do not submit a pull requet without
unit tests. Test can be run from the `baibai-trader` directory with
`nosetests tests/`.

When developing new algorithms, it is helpful to validate their performance
against past market data. If you gather some market data using the
`DummyAuthenticator` or `PracticeAuthenticator`, you pass that log file to an
instance of `AlgorithmValidator` to see how your new algorithm would have
performed had it been running at that time. See `validate.py` for a full
example.

```python
# ...

# Run validation and plot the results
validator = AlgorithmValidator(price_log, algorithm, holdings, balance)
validator.simulate_trading()
plot_pairs = validator.data_pairs_for_plotting()

px = plot_pairs['prices']['dates']
py = plot_pairs['prices']['values']
plt.plot(px, py, label='price')

bx = plot_pairs['buys']['dates']
by = plot_pairs['buys']['values']
plt.plot(bx, by, '*', label='buys')

sx = plot_pairs['sells']['dates']
sy = plot_pairs['sells']['values']
plt.plot(sx, sy, 'o', label='sells')
plt.show()
```
