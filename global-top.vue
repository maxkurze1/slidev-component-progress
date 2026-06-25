<template>
  <!-- <Progress bar-color="blue" thickness="10px" clicks fillLast offset="0px"/> -->
  <Progress position="bottom" bar-color="#ff8054" level="2" print fade>
    <template #marker="args"><MarkersShapes v-bind="args" nested activeStyle="filled"/></template>
  </Progress>
  <Progress bar-color="#0099ff" level="3" clicks :opacity="1" print marginY="0px" fillLast emptyFirst scale="clicks" marginL="100px">
    <template #marker="args"><MarkersShapes v-bind="args" shape="square" visible="hover" /></template>
  </Progress>
</template>
