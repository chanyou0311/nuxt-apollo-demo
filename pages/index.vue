<template>
  <div>
    <!-- テーブル上部に登録ボタンを設置します。 -->
    <button @click="isShowEditArea = true">
      新規登録
    </button>

    <!-- テーブルに編集ボタンと削除ボタンの列を追加します。 -->
    <table>
      <tr>
        <th>ID</th>
        <th>名前</th>
        <th>メール</th>
        <th>年齢</th>
        <th>-</th>
      </tr>

      <tr v-for="item in users" :key="item.id">
        <td>{{ item.id }}</td>
        <td>{{ item.name }}</td>
        <td>{{ item.email }}</td>
        <td>{{ item.age }}</td>
        <td>
          <button @click="editItem(item)">
            編集
          </button>
          <button @click="deleteItem(item)">
            削除
          </button>
        </td>
      </tr>
    </table>

    <!-- 登録と更新の両方で使う編集エリアです。 -->
    <!-- 登録か更新かの判定はeditedIndexを見ます。 -->
    <div v-if="isShowEditArea">
      <h2>{{ formTitle }}</h2>
      <div>
        <input v-model="editedItem.name" type="text" placeholder="名前" />
      </div>
      <div>
        <input v-model="editedItem.email" type="email" placeholder="メール" />
      </div>
      <div>
        <input
          v-model.number="editedItem.age"
          type="number"
          placeholder="年齢"
        />
      </div>
      <div>
        <button @click="close">
          キャンセル
        </button>
        <button @click="save">
          保存
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import getUsersGql from "~/apollo/queries/getUsers.gql";
// 今回定義したgqlファイルをインポートします。
import createUserGql from "~/apollo/mutations/createUser.gql";
import updateUserGql from "~/apollo/mutations/updateUser.gql";
import deleteUserGql from "~/apollo/mutations/deleteUser.gql";

import userCreatedGql from "~/apollo/subscriptions/userCreated.gql";
import userUpdatedGql from "~/apollo/subscriptions/userUpdated.gql";
import userDeletedGql from "~/apollo/subscriptions/userDeleted.gql";

export default {
  data() {
    return {
      users: [],

      // 登録・更新用の編集エリア表示非表示のためのフラグです
      isShowEditArea: false,

      // 登録か更新かを判定するための値です。
      // -1の場合は登録
      // それ以外の場合は更新　と判定します。
      editedIndex: -1,

      // 編集エリア用のモデルです。
      editedItem: {
        id: null,
        name: null,
        email: null,
        age: null
      },

      // 編集エリア用のモデルを初期化用モデルです。
      // 初期化時に
      // this.editedItem = Object.asigne({}, this.defaultEditedItem)
      // のように使います。
      defaultEditedItem: {
        id: null,
        name: null,
        email: null,
        age: null
      }
    };
  },

  // 更新の場合、更新対象が何番目かをeditedIndexにセットしているので
  // editedIndexの値によって、更新か登録かを判断し
  // 編集エリアのタイトルを出しわけます。
  computed: {
    formTitle() {
      return this.editedIndex === -1 ? "ユーザー新規登録" : "ユーザー更新";
    }
  },

  apollo: {
    users: {
      query: getUsersGql, // SubscriptionはSmart QueryのsubscribeToMoreで指定します。
      subscribeToMore: [
        // 登録時の処理です
        {
          document: userCreatedGql,
          // Subscription発生時の処理をupdateQueryに定義します。
          // 第一引数は前回のusers
          // 第二引数はサーバーからのレスポンス情報　です。
          updateQuery: (prev, { subscriptionData }) => {
            if (!subscriptionData.data) {
              return prev;
            }

            const newUser = subscriptionData.data.userCreated;
            // ここで返した値がusersに代入されます。
            return prev.users.push(newUser);
          }
        },

        // 更新時の処理です。
        // 登録時とほぼ同様の処理なので説明は割愛します。
        {
          document: userUpdatedGql,
          updateQuery: (prev, { subscriptionData }) => {
            if (!subscriptionData.data) {
              return prev;
            }

            const updatedUser = subscriptionData.data.userUpdated;
            const targetUser = prev.users.find(
              user => user.id + "" === updatedUser.id + ""
            );
            targetUser.name = updatedUser.name;
            targetUser.email = updatedUser.email;
            targetUser.age = updatedUser.age;
            return prev.users;
          }
        },

        // 削除時の処理です。
        // 登録時とほぼ同様の処理なので説明は割愛します。
        {
          document: userDeletedGql,
          updateQuery: (prev, { subscriptionData }) => {
            if (!subscriptionData.data) {
              return prev;
            }

            const deletedUser = subscriptionData.data.userDeleted;
            const userIndex = prev.users.findIndex(
              user => user.id + "" === deletedUser.id + ""
            );

            if (userIndex === -1) throw new Error("User not found");

            prev.users.splice(userIndex, 1);
            return prev.users;
          }
        }
      ]
    }
  },

  methods: {
    //登録処理です。
    async createItem({ name, email, age }) {
      // mutationを発行する場合は、Smart Queryではなく
      // $apolloを使って処理します。
      const { error } = await this.$apollo.mutate({
        mutation: createUserGql,
        variables: {
          name,
          email,
          age
        }
      });

      if (error) {
        // エラー処理
      }

      // 編集エリアを非表示にします。
      this.close();
    },

    // 更新処理です。
    // 登録処理とほぼ同様の処理なので説明は割愛します。
    async updateItem({ id, name, email, age }) {
      const { error } = await this.$apollo.mutate({
        mutation: updateUserGql,
        variables: {
          id,
          name,
          email,
          age
        }
      });

      if (error) {
        // エラー処理
      }

      this.close();
    },

    // 削除処理です。
    // 登録処理とほぼ同様の処理なので説明は割愛します。
    async deleteItem(item) {
      if (!confirm(`ユーザー(${item.name})を削除しますか?`)) {
        return;
      }

      const { error } = await this.$apollo.mutate({
        mutation: deleteUserGql,
        variables: {
          id: item.id
        }
      });

      if (error) {
        // エラー処理
      }
    },

    // 一覧の編集ボタンクリック時の処理です
    // 編集エリアの変数に値を代入して、編集エリアを表示します。
    editItem(item) {
      this.editedIndex = this.users.indexOf(item);
      this.editedItem = Object.assign({}, item);
      this.isShowEditArea = true;
    },

    // 編集エリアを非表示にする処理です。
    // 編集エリアのモデルの初期化もあわせて実施します。
    close() {
      this.isShowEditArea = false;
      setTimeout(() => {
        this.editedItem = Object.assign({}, this.defaultEditedItem);
        this.editedIndex = -1;
      }, 300);
    },

    // 編集エリアで保存ボタンをクリックした場合の処理です。
    // editedIndexをみて登録か更新かを判断します。
    save() {
      if (this.editedIndex > -1) {
        this.updateItem(this.editedItem);
      } else {
        this.createItem(this.editedItem);
      }
    }
  }
};
</script>
