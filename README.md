
##### Example E2E testing from API

```typescript


import { Transaction, Block, Account, Unit, TransactionReceipt, Tx, Personal } from '../../core/services/contract/web3.types';
import { IQuery, IMutation } from '../../../api-types/graphql';
import { AtcTestUtil } from '../../core/testing/test.util';
import { User } from '../../core/models/User';
import { SequelizeController } from '../../core/sequelize.controller';
import { Wallet } from '../../core/models/Wallet';
import { EthereumAPI } from './ethereum.api';
import { AuthModule } from '../../core/services/auth/auth.module';
import { AtcPerson } from './services/person.service';
import { BUY_COINS_MUTATION } from '../../core/testing/mutations/buyCoins.mutation';
import { SEND_TRANSACTION_MUTATION } from '../../core/testing/mutations/sendTransaction.mutation';
import { ServerErrorsList } from '../../core/services/errors/server-errors-list';

const atcTestUtil: AtcTestUtil = new AtcTestUtil();

beforeAll(() => atcTestUtil.init());
afterAll(() => atcTestUtil.destroy());

describe('Ethereum Controller', () => {

  // tslint:disable-next-line:max-line-length
  it(`e2e: mutation -> (buyCoins) : Should sucessfully buy ${1000 * Number(atcTestUtil.defaultTestingAmount)} ATC coin with ${atcTestUtil.defaultTestingAmount} ETH`, async done => {
    atcTestUtil.sendRequest<IMutation>({
      query: BUY_COINS_MUTATION,
      variables: {
        address: atcTestUtil.users.USER.wallets[0].address,
        amount: atcTestUtil.defaultTestingAmount
      },
      signiture: atcTestUtil.users.USER
    })
      .subscribe(async res => {
        if (!res.success) {
          console.error(res.errors[0].name);
        }
        expect(res.success).toBeTruthy();
        expect(res.data.buyCoins.tx).toBeDefined();
        done();
      }, err => {
        expect(err).toBe(null);
        done();
      });
  });

  it(`e2e: mutation -> (buyCoins) : Should throw error 'invalid-address'`, async done => {
    atcTestUtil.sendRequest<IMutation>({
      query: BUY_COINS_MUTATION,
      variables: {
        address: 'g',
        amount: atcTestUtil.defaultTestingAmount
      }
    })
      .subscribe(res => {
        expect(res.success).toBeFalsy();
        expect(res.errors[0].name).toBe('invalid-address');
        done();
      }, err => {
        expect(err).toBe(null);
        done();
      });
  });

  it(`e2e: mutation -> (buyCoins) : Should throw error 'missing-wallet-id' when buying ATC Coins`, async done => {
      atcTestUtil.sendRequest<IMutation>({
        query: BUY_COINS_MUTATION,
        variables: {
          address: '0xB1dCf5215d8f537F47B286699746Bf696A768bbc',
          amount: atcTestUtil.defaultTestingAmount
        }
      })
        .subscribe(res => {
          expect(res.success).toBeFalsy();
          expect(res.data.buyCoins).toBeNull();
          expect(res.errors[0].name).toBe('missing-wallet-id');
          done();
        }, err => {
          expect(err).toBe(null);
          done();
        });
    });

  it(`e2e: mutation -> (sendTransaction) : Should throw error 'account-not-found' when sending transaction from unknown account`, done => {
    atcTestUtil.sendRequest<IMutation>({
      query: SEND_TRANSACTION_MUTATION,
      variables: {
        from: '0x67De18cA13BDd44Bed2ACF1062866E93FB9F479B',
        to: '0xE3b6f16234Ac91d5Cd5A1030d13839a8be671b06',
        amount: atcTestUtil.defaultTestingAmount
      }
    })
      .map(res => {
        expect(res.success).toBeFalsy();
        expect(res.data.sendTransaction).toBeNull();
        expect(res.errors[0].name).toBe('account-not-found');
        done();
      }, (err) => {
        expect(err).toBe(null);
        done();
      }).subscribe();
  });

});

```



##### Example Unit testing

```typescript

import { Block, Account, Unit, TransactionReceipt, Tx, Personal } from '../../core/services/contract/web3.types';
import { IQuery, IMutation } from '../../../api-types/graphql';
import { AtcTestUtil } from '../../core/testing/test.util';
import { User } from '../../core/models/User';
import { SequelizeController } from '../../core/sequelize.controller';
import { Wallet } from '../../core/models/Wallet';
import { FAKE_USERS_WALLETS_COUNT } from '../../core/testing/fakeUsers';
import { EthereumAPI } from './ethereum.api';
import { AuthModule } from '../../core/services/auth/auth.module';
import { AtcPerson } from './services/person.service';
import { Transaction } from '../../core/models/Transaction';

const atcTestUtil: AtcTestUtil = new AtcTestUtil();

beforeAll(() => atcTestUtil.init());
afterAll(() => atcTestUtil.destroy());

describe('Ethereum Api', () => {

  it(`unit: (FakeWalletsCount) : Should have ${FAKE_USERS_WALLETS_COUNT.count} wallets inside personal web3`, async done => {
    expect(await atcTestUtil.getEtherAccounts()).toHaveLength(FAKE_USERS_WALLETS_COUNT.count);
    done();
  });

  // tslint:disable-next-line:max-line-length
  it(`unit: (sendTransaction) : Should send transaction from USER to ADMIN ${atcTestUtil.defaultTestingAmount} ETH`, async done => {
    const person = new AtcPerson(
      AuthModule.decrypt(atcTestUtil.users.USER.credential.password),
      atcTestUtil.users.USER.wallets[1].address
    );
    await person.unlockAccount();
    EthereumAPI.sendTransaction(
      atcTestUtil.users.USER.wallets[1].address,
      atcTestUtil.users.ADMIN.wallets[1].address,
      atcTestUtil.defaultTestingAmount,
      atcTestUtil.users.USER.credential,
      null,
      null
    ).then(async transaction => {
      expect(transaction.blockHash).toBeTruthy();
      const t = await Transaction.find({where: {hash: transaction.transactionHash}});
      expect(transaction.transactionHash).toBe(t.hash);
      await t.destroy();
      await person.lockAccount();
      done();
    }).catch(e => {
      expect(e).toBe(null);
      done();
    });
  });

  // tslint:disable-next-line:max-line-length
  it(`unit: (sendTransaction) : Should throw error invalid-account-password when sending from ADMIN to USER ${atcTestUtil.defaultTestingAmount} ETH`, async done => {
    EthereumAPI.sendTransaction(
      atcTestUtil.users.ADMIN.wallets[1].address,
      atcTestUtil.users.USER.wallets[1].address,
      atcTestUtil.defaultTestingAmount,
      atcTestUtil.users.ADMIN.credential,
      null,
      null
    ).then(async transaction => {
      expect(transaction.blockHash).toBeTruthy();
      done();
    }).catch(e => {
      expect(e.name).toBe('invalid-account-password');
      done();
    });
  });

});


```