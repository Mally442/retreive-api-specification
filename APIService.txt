using BLL.Models.Billing;
using BMS.Clients;
using BMS.Helpers;
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using System.Net.Http;

namespace BLL.Services.Domain.Billing
{
	public interface IAPIService
	{
		IEnumerable<RemoteAPI> GetAllRemoteAPIs(BLL.Lookups.Billing.APIType type, bool cache = true);
	}

	public class APIService : IAPIService
	{
		private const string CacheKey = "BLL.Services.Domain.Billing.APIService.RemoteAPIs_{0}";

		public IEnumerable<RemoteAPI> GetAllRemoteAPIs(BLL.Lookups.Billing.APIType type, bool cache = true)
		{
			string cacheKey = APIService.CacheKey.FormatText(type);

			var result = new RemoteAPI[0];
			if (cache && Context.Cache.ContainsKey(cacheKey))
			{
				result = Context.Cache.Get<RemoteAPI[]>(cacheKey);
			}
			else
			{
				var task = Task.Run(async () =>
				{
					using (var client = Resolver.Get<APIsClient>())
					{
						return await client.FetchAsync(type.ToInt()).ConfigureAwait(false);
					}
				});
				task.Wait();

				if (task.Result.IsSuccessStatusCode)
				{
					var contentTask = Task.Run(async () => { return await task.Result.Content.ReadAsAsync<RemoteAPI[]>().ConfigureAwait(false); });
					contentTask.Wait();
					result = contentTask.Result;
				}
				Context.Cache.Set(cacheKey, result, DateTime.Now.AddHours(1));
			}
			return result;
		}
	}
}
