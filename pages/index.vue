<template>
  <table>
    <tr>
      <th>ID</th>
      <th>名前</th>
      <th>メール</th>
      <th>年齢</th>
    </tr>
    <tr v-for="item in users" :key="item.id">
      <td>{{ item.id }}</td>
      <td>{{ item.name }}</td>
      <td>{{ item.email }}</td>
      <td>{{ item.age }}</td>
    </tr>
  </table>
</template>

<script>
import getUsersGql from "~/apollo/queries/getUsers.gql";

export default {
  data() {
    return {
      users: []
    };
  },

  apollo: {
    users: {
      query: getUsersGql
    }
  }
};
</script>
