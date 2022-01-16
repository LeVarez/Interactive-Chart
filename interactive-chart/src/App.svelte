<script>
	import data from './data/data.json';

	import locale from '@reuters-graphics/d3-locale';
	import InteractiveChart from './InteractiveChart.svelte';

	export let color1 = '#5cc6b2';
	export let color2 = 'red';


	const loc = new locale('es');

	//Axis format
	const format = {
		x: loc.formatTime('%Y'),
        y: loc.format(',.1d')
    }
</script>


	<div class="test">
		<InteractiveChart
			data={data}
			key={{x: 'x', y: 'y'}}
			{format}
			color = {[color1, color2]}
		/>
	</div>



<style>
	.test {
		width: 75%;
		height: 50%;
	}
</style>